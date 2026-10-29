# Kotlin 协程与 Flow（面试主战场）

## 一、协程本质：状态机 / CPS，而不是线程

### 问题

`suspend` 函数编译后变成了什么？"协程就是切线程"错在哪里？

### 机制级答案

- **CPS 变换**：`suspend fun foo(): T` 编译后形态接近 `foo(continuation: Continuation<T>): Object`——返回值参与挂起协议；`COROUTINE_SUSPENDED` 标记"当前未完成，将来由 continuation.resumeWith 恢复"。
- **状态机**：函数体中每个**挂起点**（调用其他 suspend、挂起返回）对应状态编号；恢复时 `when (state)` 从断点续跑。跨挂起存活的局部变量被编译器保存（装入状态机对象/寄存器结构）——这是"协程局部变量有生命周期"的编译根源。
- **与线程的关系**：挂起**不占用线程**；恢复通过 `ContinuationInterceptor`（即 CoroutineContext 中的 Dispatcher）决定在哪个线程执行状态机下一段。`Dispatchers.Main` 用主线程 Handler 派发，`IO`/`Default` 绑线程池——**线程切换是 Dispatcher 的职责，不是 suspend 的本质**。
- **阻塞 vs 挂起**：`Thread.sleep` 阻塞线程；`delay` 挂起协程并让出线程、由定时器恢复。误用阻塞 API 于 Default/IO 池会占住池线程，拖垮并发度。
- **单线程并发**：同一 Dispatcher 线程上多个协程按派发队列交错执行——没有 await 点就不会交错；**挂起前后不存在隐式原子性**（见 Coroutines/4.md 共享可变状态）。

### 【为什么这样设计】

把"并发逻辑"从"线程栈"解耦：栈不可续，而显式 continuation + 状态机可序列化、可调度、可超时、可结构化——用少量分配换海量逻辑并发；CPS 是在 JVM 不改指令集前提下实现可恢复调用链的标准手段。

### 【面试官视角：考察点 / 加分 / 减分】

- 考察点：CPS/状态机、COROUTINE_SUSPENDED、Dispatcher 与 suspend 的职责边界、阻塞 vs 挂起。
- 加分：提到局部变量入状态对象；提到 `intercepted()`/`resumeWith`；能把 `withContext` 解释为"另一个协程完成回调 + 状态机切换"。
- 减分：说"suspend 会暂停线程"；把协程默认当成更轻量的线程池线程数量问题。

### 【追问链】

1. 为什么每个挂起点后局部变量都要假设可能被保存？→ 编译器保守保存；热路径优化看逃逸分析（编译器内部）。
2. suspend 内调用 Java 阻塞 API 会怎样？→ 阻塞 Dispatcher 线程；需 `withContext(IO)` 或专门 Dispatcher。
3. 与虚拟线程的关系？→ 逻辑并发模型不同：虚拟线程仍是"一线程一栈"由 JVM 调度；协程是语言级 CPS。可共存，不互相替代（见 `../Java/Senior/JVMEngineering` 对照）。

---

## 二、CoroutineContext：Job / Dispatcher / 拦截器

### 问题

CoroutineContext 由什么组成？元素如何合并与查找？

### 机制级答案

- **成分**：
  - **Job**：协程生命周期句柄与父子结构（见下文结构化并发）；
  - **ContinuationInterceptor**：通常即 Dispatcher（`Dispatchers.IO`、`Executor.asCoroutineDispatcher()`），拦截 resume 决定执行线程；
  - **CoroutineName / CoroutineId**：诊断；
  - **CoroutineExceptionHandler**：未捕获异常最后出口（launch 根协程）；
  - 自定义元素：`ThreadContextElement` 等可把值注入线程局部（如 MDC）。
- **不可变集合语义**：Context 是 key-value 的不可变组合；`+` 合并规则——同 key 覆盖，Job 与 Dispatcher 分开叠加；`coroutineContext` 可在协程内读取。
- **继承**：子协程继承父 launch/async 时传入的 context **与默认父元素合并**；显式传入 `Job()` 会**替换**继承 Job（打破结构化，慎用）；`SupervisorJob()` 作为替换时改变失败传播（见结构化）。
- **ThreadLocal 陷阱**：Dispatcher 线程复用导致 ThreadLocal 串值；协程侧应用 `ThreadContextElement` 或 `asContextElement` 包装。

### 【为什么这样设计】

把"生命周期、调度、命名、错误"从单一继承体系拆成正交元素，用合并代数组合——新增能力不改协程核心；不可变合并避免并发修改 context。

### 【面试官视角：考察点 / 加分 / 减分】

- 考察点：Job vs Dispatcher 角色、`+` 覆盖、替换 Job 的后果、ThreadLocal。
- 加分：`asContextElement` 恢复 MDC；提到 `CoroutineDispatcher.limitedParallelism`（版本演进中，语义级）。
- 减分：把 context 当 Map 随便 put；新建 `Job()` 当作"更安全的默认"。

### 【追问链】

1. `launch` 默认继承什么？→ 几乎全部（含 Job 作为父）、Dispatcher 有默认 `Default` 除非 scope 指定。
2. 换 Dispatcher 只能 withContext？→ 或 launch 时指定 context/`flowOn` 管 Flow 上游。
3. `coroutineContext[Job]` 用途？→ 取消句柄、join、作为 `asCoroutineDispatcher` 无关问题时仍常用于结构化。

---

## 三、launch / async / runBlocking / withContext 语义边界

### 问题

四个最常用构建器在**返回值、异常、调度、阻塞**四个维度如何区分？

### 机制级答案

| 构建器 | 返回 | 异常默认 | 是否阻塞调用线程 | 典型用途 |
|--------|------|----------|------------------|----------|
| `launch` | Job | 立即传播父（root 可 handler） | 否 | 火即忘、并行子任务 |
| `async` | Deferred | 挂起至 await 才抛 | 否 | 需要结果/结构化并发汇总 |
| `runBlocking` | 阻塞结果 | 可在边界 try/catch 或传播 | **是** | 桥接非挂起环境、测试 |
| `withContext` | lambda 返回值 | 同步于当前协程语义，失败使当前协程失败 | 否（挂起等待） | 切 Dispatcher、隔离环境 |

- **launch vs async**：不接 async 的 await 会把异常变成未捕获（或仅在未处理时）——**async 必须 await 或等价消费**；launch 可以在顶层装 handler。
- **withContext**：不是新"任务提交后立即返回"——它挂起等待块完成；块内失败默认让调用方协程失败；可换 Job 吗？传入不同 Job 危险；主要用于 Dispatcher/上下文隔离。`withContext(NonCancellable)` 在取消后仍执行清理（见第2页）。
- **runBlocking**：同步阻塞桥；主线程/单线程 UI 事件循环里嵌套长阻塞会卡 UI；单元测试可接受；**禁止**在 suspend API 内部再 runBlocking 造成潜在死锁（嵌套事件循环语义陷阱）。
- **CoroutineScope**：自建 scope 需明确 Job 策略与 Dispatcher；`MainScope()` 提供 SupervisorJob + Main，记得 cancel。

### 【为什么这样设计】

四者覆盖"提交、取结果、阻塞桥、环境切换"四种并发语法；刻意分离 Job 返回与 Deferred 返回，使结构化树上每条边的失败策略可单独选择。

### 【面试官视角：考察点 / 加分 / 减分】

- 考察点：async 必 await、withContext 同步等待、runBlocking 适用域、launch 默认失败。
- 加分：`coroutineScope { }` 与 async 组合的结构化汇总；`supervisorScope` 差异；提到 `runInterruptible` 等边界工具的思路。
- 减分：在 Android 主线程 runBlocking 做 IO；用 GlobalScope.async 替代 withContext。

### 【追问链】

1. `coroutineScope` 与 `withContext`？→ coroutineScope 建子作用域等待全部子 Job，失败 fail-fast；不换 Dispatcher。
2. async 在取消时？→ 子被取消，await 抛 CancellationException（可恢复语义见第2页）。
3. 如何限制并发度？→ Semaphore、limitedParallelism（以所用版本 API 为准）、或自定义 Dispatcher。

---

## 四、结构化并发：父子 Job、取消传播、Supervisor

### 问题

什么是结构化并发？父子失败如何传播？SupervisorJob 改变了什么？

### 机制级答案

- **树约束**：每个协程有父 Job；子协程完成前父不完成；父取消 → 所有子取消；子失败（非 Cancellation）→ 默认**父失败 + 所有兄弟取消**（fail-fast），防止"部分失败的幽灵子树"长期存在。
- **取消传播**：取消是**协作式**信号沿树向下；不自动中断阻塞代码（见第2页）。
- **SupervisorJob / supervisorScope**：子失败**不**传染父与兄弟；各子失败独立，需各自 handler 或 await 处理。用于"多个独立子任务互不拖垮"（UI 多请求、多子组件）。
- **作用域 API**：`coroutineScope` 建立 fail-fast 子作用域；`supervisorScope` 同结构但失败隔离；`produce`/`actor` 等在此模型上扩展（Channel 相关见第4页）。
- **全局作用域**：`GlobalScope` 无父 → 无结构化、不随页面取消 → 泄漏与"页面销毁任务仍在写 View"经典事故；生产禁用，除非进程级单例任务且有明确生命周期。

### 【为什么这样设计】

结构化并发把并发树与生命周期树对齐：**作用域死则子树死**，用作用域词汇表达"这批任务属于这个页面/这个请求"——解决线程时代"谁负责取消"的责任真空。

### 【面试官视角：考察点 / 加分 / 减分】

- 考察点：fail-fast 默认、supervisor 边界、GlobalScope 之害、作用域与 UI 生命周期对齐。
- 加分：讲清"子 Job 失败先取消兄弟再通知父"的级联；自建 scope 必有 cancel 测试；多模块共享 scope 的设计。
- 减分：到处 SupervisorJob"防崩"却没人处理异常；用 GlobalScope 做页面请求。

### 【追问链】

1. 启动模式 DEFAULT/LAZY/ATOMIC/UNDISPATCHED 差异？→ LAZY 首 await/join 才启动；UNDISPATCHED 立即在当前线程跑至下一挂起点；按需选。
2. 父在子未完成时 cancel 了自己？→ 子收到取消，父等待子结束才完成取消（保证资源回收）。
3. Supervisor 下异常去哪？→ 传播到该子的 handler 或调用方 await；父仅记为已完成不失败。

---

## 一、取消协作：isActive / CancellationException / finally / NonCancellable

### 问题

为什么"取消"不会立刻停下协程？正确的取消响应写法有哪些坑？

### 机制级答案

- **协作模型**：`job.cancel()` 只是**标记取消**并通知子树；协程在**下一个挂起点**检查取消并抛 `CancellationException`。CPU 密集循环必须自行检查 `ensureActive()` / `isActive` 或在挂起点被动响应——**不检查挂起点的忙循环取消不生效**。
- **取消即异常**：挂起点抛出 `CancellationException`（CE）；协程机制把它当**正常取消路径**，不当失败向上传播到 handler。**catch (e: Exception) 吞掉 CE 会导致协程无法真正停止**——正确做法：
  - `catch (e: CancellationException) { throw e }` 重新抛出；
  - 或只捕获特定业务异常类型。
- **finally 与挂起**：取消后 finally 中若再调用挂起函数，若作用域仍在取消中会**再次抛 CE**；释放资源/上报收尾要用 `withContext(NonCancellable) { ... }` 包裹——显式声明"清理不可取消"。
- **资源关闭**：`use`/`try/finally` 关闭 Channel、连接；`Job.cancelAndJoin` 等待真正结束——"cancel 返回不等于完成"。
- **阻塞 API**：`runInterruptible`（概念：包装阻塞调用响应中断）；`Thread.interrupt` 语义要真能被底层响应。

### 【为什么这样设计】

协作式取消避免在任意字节码点安全点式中断（与线程 stop 类比的危险）——把取消变成可推理的异常协议，清理路径用 NonCancellable 显式化，牺牲一点"自动停"换正确性与可测试性。

### 【面试官视角：考察点 / 加分 / 减分】

- 考察点：挂起点检查、CE 不可吞、NonCancellable、cancel ≠ 完成。
- 加分：循环里 `ensureActive()`；finally 清理模板；提到取消后 resume 与 CE 的状态机一致语义。
- 减分：`catch (e: Exception) {}` 吃掉取消；认为 cancel 立即停线程。

### 【追问链】

1. catch CE 后还能恢复协程吗？→ 协程处于取消路径，恢复需新协程；不要在 CE 后强行"继续业务"。
2. `withContext(NonCancellable)` 死锁？→ 不换 Dispatcher 时在当前 Dispatcher 跑清理；长清理仍会占线程。
3. Job 取消后 join？→ `job.join` 挂起至完成；对已取消 job join 正常返回完成态。

---

## 二、异常传播矩阵：launch vs async vs withContext

### 问题

同样的业务异常，在 launch、async、withContext、Flow 中路径有何不同？谁会触发 CoroutineExceptionHandler？

### 机制级答案

- **矩阵（普通 Exception，非 CE）**：

| 边界 | 传播行为 | 谁能 catch | handler？ |
|------|----------|------------|-----------|
| `launch` 根协程 | 立即上抛至父；根上交给 handler | 同协程内 try/catch 可捕业务异常（含跨挂起点抛出的）；子 launch 的失败走 Job 树、不经该 try | 根 launch 用 CEH |
| `launch` 子协程 | 先取消兄弟与父（fail-fast） | 父可用 `join`+ 取消原因或 supervisor 分离 | 子协程装 CEH **无效**（先上父） |
| `async` 根 | 存入 Deferred，await 时抛 | 调用方 try/catch await | 根 async 默认**不上 CEH**，未 await 变 uncaught |
| `async` 子 + 父普通 Job | 与 launch 同，失败级联 | — | supervisor 下独立 |
| `withContext` | 块内异常使**当前协程**失败（像同步抛出） | 调用方同一协程 try/catch 有效 | 不单独装 CEH |
| `Flow.collect` 内 | 向上抛给收集协程 | try/catch 围住 collect 或操作符 | 同收集协程规则 |

- **关键规则**：
  1. 异常沿 Job 树传播，**CEH 只安装在"根"**（scope.launch/async 最外层）；
  2. `async` 延迟抛出——在**无父等待**（如 GlobalScope）时必须 await，否则 uncaught；在 `coroutineScope` 内子失败会使 scope 失败并重新抛出，await 影响观察时机与结果获取；
  3. try/catch 可围住**同一协程**内调用链抛出的业务异常；**不能**替代对子 launch 失败的结构化处理（子失败级联父，不在子调用点的 try 内）；
  4. `supervisorScope`/SupervisorJob 子失败不拉父，每个子自己负责 handler 或 await。

### 【为什么这样设计】

"立即传播"保护结构化树不留半失败状态；"延迟 await"允许组合多个计算为一个结果。CEH 与父子机制分工：结构化管级联，handler 管兜底上报——避免每层 try/catch 模板。

### 【面试官视角：考察点 / 加分 / 减分】

- 考察点：矩阵默写能力、async 未 await、子协程 CEH 无效、coroutineScope 汇总。
- 加分：把 CEH 与 `Thread.setDefaultUncaughtExceptionHandler`/崩溃 SDK 对接；Flow 收集异常单独讨论。
- 减分：以为 async 崩溃只在 await；在子协程里装 CEH 认为能拦父传播。

### 【追问链】

1. `coroutineScope` 内 launch 失败？→ scope 失败，外层协程跟着失败；外层 catch 可捕。
2. Flow 里 launch 的异常？→ 属收集 scope 的子任务，按 launch 规则级联。
3. 多个 async 失败几个？→ 智能取消其余，awaitAll 先抛者为准，其余挂在取消树。

---

## 三、CoroutineExceptionHandler 边界

### 问题

CEH 的安装位置、捕获范围、与崩溃治理如何衔接？

### 机制级答案

- **安装**：`context + CoroutineExceptionHandler { _, e -> … }` 或 `launch(handler)`；仅当异常到达**该协程作为"根"且未被父接管结构化失败路径**时调用。子协程中的 handler 在父 Job 传播规则下往往收不到普通异常（父先失败取消）。
- **捕获内容**：普通异常 +（根上）未处理失败；**不**替代 try/catch 业务逻辑；**不**用于"吞掉崩溃继续运行"——进程状态不可信时与 Java 崩溃治理同理（旁路观测优先）。
- **Supervisor 下**：每个独立子需自己的 handler 或 await——"多请求隔离"模板：`scope.launch(handler) { launch { A }… }` 结构要小心；更常见是每个子任务 `supervisorScope`/各自根 launch + handler。
- **Android 衔接**：主线程未处理异常仍会走 Android 默认 handler；CEH 中上报后**不要吞掉** CE 语义导致状态机错乱——上报完让异常继续或按产品定义重启策略。

### 【为什么这样设计】

CEH 是结构化树的"兜底观测点"，不是业务错误通道——业务错误应在类型上（Result/sealed）或 await 处理；把两者混用会把可恢复错误升级成协程失败。

### 【面试官视角：考察点 / 加分 / 减分】

- 考察点：根安装、不吞业务错误、与崩溃 SDK 关系。
- 加分：全局 scope 默认 handler 与页面 scope 策略分层；上报采样与去重（同类异常风暴）。
- 减分：给每个子 launch 都塞 handler 以为万能；handler 里 sleep/长锁。

### 【追问链】

1. handler 能恢复协程吗？→ 不能"继续原协程"；只能上报/调度新任务。
2. CE 是否触发 handler？→ 取消路径不算未捕获失败，不触发 CEH。
3. 多个根 launch？→ 各自独立根，各配 handler 或统一 scope 设计。

---

## 四、Dispatcher 选型与泄漏

### 问题

IO/Default/Main/单线程如何选？未取消的协程/全局 scope 会泄漏什么？

### 机制级答案

- **Dispatchers.Default**：CPU 密集共享池，线程数 ≈ 核数（上限有默认值）；长循环占满会饿死其他 CPU 协程。
- **Dispatchers.IO**：与 Default 共享底层池但有单独并发配额（可弹性扩展上限），适合阻塞 IO；新版本有 `IO.limitedParallelism` 与 Default 分离演进（语义级：把阻塞任务隔离出 CPU 池），细节随版本文档。
- **Dispatchers.Main / Main.immediate**：Android 主线程；immediate 在已是主线程时不经过 dispatch 直接执行（减少一次 post），UI 更新友好。
- **单线程 dispatcher**：`newSingleThreadContext`/Executors 单线程 → 串行化简单模型，但**记得关闭**否则线程泄漏；现代替代：`limitedParallelism(1)` 或 Mutex 串行（第4页）。
- **泄漏形态**：
  - GlobalScope/手工 scope 未 cancel：任务持有 Activity/View 引用 → 内存泄漏 + 更新已销毁 UI；
  - Dispatcher 线程池常驻：测试未替换为 TestDispatcher → 单测泄漏与异步不稳；
  - `newSingleThreadContext` 不 close → 线程数增长。
- **治理**：所有页面级 scope 与生命周期对齐（Android/1.md）；进程级任务用单例 scope + 显式生命周期；CI 测试检查泄漏（LeakCanary + 协程完成回调）。

### 【为什么这样设计】

Dispatcher 抽象把"池策略"从业务代码剥离；IO 与 Default 分离是对"阻塞饿死计算"的事故驱动设计；泄漏根因几乎总是**作用域生命周期长于引用对象**——结构化并发给出对齐工具，不用好就是高级版线程泄漏。

### 【面试官视角：考察点 / 加分 / 减分】

- 考察点：IO vs Default、Main.immediate、单线程 close、GlobalScope 危害。
- 加分：limitedParallelism 做并发闸门；自定义 Executor dispatcher 的背压意识；测试替换 dispatcher。
- 减分：所有 IO 都 Dispatchers.Default；无 close 的单线程上下文；泄漏只怪 JVM 不提作用域。

### 【追问链】

1. IO 和 Default 能互相饿死吗？→ 需按版本池模型理解；工程上仍应用 limitedParallelism/分池隔离阻塞。
2. 如何给 OkHttp 同步调用封装？→ withContext(IO) + 超时；不建议每请求新建线程。
3. Android 主线程安全？→ Main + 避免在协程内触摸已销毁 UI（生命周期 scope 解决）。

---

## 一、冷流与 collect 惰性

### 问题

Flow 为什么是"冷"的？`flowOn` 改变的是谁的线程？

### 机制级答案

- **冷语义**：`flow { emit(...) }` 在**没有 collect 前不执行**；每次 collect 重新运行生产者——与 Sequence 同构。副作用型 Flow（发请求）在多次 collect 时会重复执行，除非热化（见下节）。
- **收集即驱动**：生产者在 collect 协程的上下文中逐步运行；每个 `emit` 下游处理完才返回（无隐式缓冲）——天然**反压到生产速度**。
- **flowOn**：只影响**上游**操作符的执行上下文（生产线程），不影响下游 collect 线程；它通过在边界插入 channel 分发实现（因此会引入异步边界与缓冲语义，默认缓冲 64，以版本为准可配 `buffer`）。
- **取消**：collect 所在协程取消时，Flow 在下一个挂起点停止；生产者需尊重取消（IO 调用选可取消 API）。
- **异常**：生产者/操作符抛出普通异常 → 传播给收集协程（见矩阵）；`catch` 操作符捕获**上游**异常（在 flowOn 边界内）；`onCompletion` 无论成败可见取消/结束原因（概念级）。

### 【为什么这样设计】

冷流把"数据源描述"与"消费"解耦，使操作符可复用、可重放；无隐式并发缓冲避免隐藏队列失控——需要吞吐时显式 `buffer`/合并操作符，把性能决策留在代码可见处。

### 【面试官视角：考察点 / 加分 / 减分】

- 考察点：每次 collect 重跑、flowOn 仅上游、emit 无缓冲、与 Sequence 类比。
- 加分：`flow {}` 内不安全 emit 的上下文规则（单线程顺序）；提到 buffer 与 channelOverflow 的显式选择。
- 减分：把 Flow 当 LiveData 默认每次只订一次；用 GlobalScope 内部 launch emit 打破冷流。

### 【追问链】

1. 为什么不能从多线程并行 emit？→ 协议要求顺序与上下文一致；并发生产用 channelFlow/共享流（版本能力语义级）。
2. flowOn 与 withContext 区别？→ withContext 换整个块上下文并等待；flowOn 是流边界且可组合在链中。
3. catch 放 flowOn 前后？→ 影响是否捕获边界异步化后的异常；原则：包住会失败的上游段。

---

## 二、冷转热：stateIn / shareIn / replay

### 问题

什么时候必须热化？`shareIn`/`stateIn` 的参数语义是什么？

### 机制级答案

- **动机**：多收集者、或"生产与收集生命周期不同步"（UI 订阅晚于数据产生、旋转重建不应重发网络）——冷流会重复副作用或丢历史。
- **SharedFlow**：`MutableSharedFlow`/`shareIn`——可多订阅；`replay`：新订阅者先收最近 N 条；`extraBufferCapacity` + `onBufferOverflow`（SUSPEND/DROP_OLDEST/DROP_LATEST）背压策略；无订阅者时是否取消上游由 `started`（Eagerly/Lazily/WhileSubscribed）决定。
- **StateFlow**：`stateIn`——总有**当前值**（必须给 `initialValue`）；订阅立即收到当前值（**粘性**）；新值基于 `equals` 去重（相同值不 emit 下游）；`WhileSubscribed` 停止上游、重订阅时若用 SharedFlow replay 可恢复——StateFlow 本身重订阅只拿当前值，不是完整历史。
- **WhileSubscribed(stopTimeoutMillis)**：最后一个订阅者离开后延迟取消上游——覆盖旋转/临时 UI 订阅；`replayExpirationMillis` 可在无订阅后清空 replay。
- **坑**：
  - StateFlow 粘性导致"订阅晚但拿到旧状态"——与 LiveData 粘性同源，UI 层消费要幂等；
  - `value = x` 相同 equals 不通知，UI 依赖"每次赋值刷新"会踩坑；
  - 在无生命周期感知的 `launch { flow.collect }` 中收集 → 页面销毁不取消（Android/1.md repeatOnLifecycle）；
  - 热流订阅者泄漏 → 上游永不取消。

### 【为什么这样设计】

热流是"广播 + 可选历史"；StateFlow 是"单槽当前值"的最小状态容器——去掉完整事件历史换简单一致性；WhileSubscribed 把生命周期控制权还给收集端，匹配 UI 订阅短暂性。

### 【面试官视角：考察点 / 加分 / 减分】

- 考察点：replay、去重、初始值粘性、WhileSubscribed、与冷流边界。
- 加分：事件用 Channel/单次 SharedFlow replay 0 避免"旋转重放"；状态用 StateFlow；背压策略选型。
- 减分：用 StateFlow 当事件总线（旋转后 toast 重放）；热流无上限 extraBuffer 导致内存。

### 【追问链】

1. 事件 vs 状态？→ 状态：StateFlow/状态机；事件：Channel 或 replay=0 SharedFlow + 生命周期消费。
2. stateIn 的 scope？→ 与热化生命周期绑定；UI 状态用 viewModelScope + WhileSubscribed 常见。
3. 相同值去重能不能关？→ StateFlow 协议固定；需要"相同值也通知"用 SharedFlow。

---

## 三、背压、buffer 与 ChannelOverflow

### 问题

Flow 默认有背压吗？`buffer` 与 `channelFlow` 如何改变模型？

### 机制级答案

- **默认背压**：无额外 buffer 时，emit 与下游处理同步——生产被自然限速；这是"协程式背压"，不同于 Rx 的请求式拉取。
- **buffer(n)**：在操作符间插入容量为 n 的 Channel 异步边界，生产者可跑在消费者前面；满时策略由 `onBufferOverflow` 决定：
  - SUSPEND：生产者挂起（保留背压，防 OOM）；
  - DROP_LATEST / DROP_OLDEST：丢事件（UI 刷新类可接受，传感器数据要按语义选）。
- **conflate**：`buffer(CONFLATED)` 近似"只保留最新"——高频状态采样；与 collectLatest 类似但边界在操作符。
- **Channel**：热的、可多发送者的并发队列；Flow 生产端多路合并（callbackFlow/ChannelFlow）最终经 channel 汇聚——Channel 是冷热之间的并发原语底座。
- **callbackFlow**：把回调 API 适配为冷流；`awaitClose` 做清理；`trySend`/`close` 尊重溢出策略——Android 回调转 Flow 标准模板。

### 【为什么这样设计】

同步 emit 把正确性默认设为"不丢不溢"；异步边界必须显式声明容量与策略——**把队列变成代码里看得见的决策**，避免无界隐式队列线上爆内存。

### 【面试官视角：考察点 / 加分 / 减分】

- 考察点：默认同步、buffer 满策略、conflate 语义、callbackFlow 清理。
- 加分：按业务选 DROP_OLDEST（最新 UI）vs SUSPEND（不可丢消息）；提到 flowOn 默认缓冲的副作用。
- 减分：到处 buffer(Unlimited)；把背压说成必须拉模型。

### 【追问链】

1. 与 RxJava 背压对比？→ Rx 请求式（request(n)）；Flow 默认推+挂起，buffer 显式。
2. 全局缓冲会不会隐藏慢消费者？→ 会，延迟与内存转移到 buffer；监控生产/消费速率。
3. Channel 与 SharedFlow？→ Channel 点对点队列；SharedFlow 多订阅广播。

---

## 四、flatMap 系列语义与异常、选型

### 问题

`flatMapConcat` / `flatMapLatest` / `flatMapMerge` 差异？Flow 与 LiveData/回调如何选？

### 机制级答案

- **外层值 → 内层流**：
  - `flatMapConcat`：顺序收集完一个内层再收下一个——不并发，背压清晰；
  - `flatMapMerge(concurrency)`：最多 N 个内层并行收集，结果按到达交错；
  - `flatMapLatest`：外层来新值时**取消**旧内层，只保留最新（搜索框防抖经典）；
  - `mapLatest`/`transformLatest` 同族。
- **异常**：任一内层失败 → 操作符失败 → 上抛收集协程；`catch` 在下游收口；取消内层用最新语义时旧内层 CE 不当失败。
- **与 LiveData 选型**：
  - LiveData：生命周期感知自动停订、主线程友好、生态熟；表达复杂流转换弱；
  - StateFlow/SharedFlow：操作符、测试（虚拟时间）、多平台；**需** repeatOnLifecycle 或显式生命周期收集防后台更新与泄漏；
  - 简单 UI 单值状态可继续 LiveData；有变换/多流合并/后台收集用 Flow。
- **与回调**：一次性结果用 suspend API；持续事件用 Flow/Channel；避免"回调里手动维护 listener 列表 + 手动反注册"双轨长期共存。

### 【为什么这样设计】

flatMap 三族对应"串行依赖、有限并发、最新胜出"三种业务模式；与 LiveData 差异是**生命周期责任模型**不同——LiveData 框架扛，Flow 用户显式扛，换灵活度。

### 【面试官视角：考察点 / 加分 / 减分】

- 考察点：Latest 的取消、Merge 并发度、LiveData 粘性/线程模型对比。
- 加分：搜索场景 flatMapLatest + debounce；列表多请求 merge + 上限；混合迁移路径。
- 减分：搜索用 flatMapMerge 不取消旧请求；宣称"Flow 完全不需要生命周期"。

### 【追问链】

1. debounce 在哪层？→ 操作符在流上，作用于外层值节奏；与 flatMapLatest 组合。
2. Flow 能替代所有 LiveData？→ 生命周期与主线程默认便利场景仍可保留；看团队约束。
3. 一次性 UI 事件坑？→ 粘性重放；用 Channel/SingleLiveEvent 类思路或状态机消费。

---

## 一、并发共享可变状态陷阱

### 问题

多个协程改同一个 `var count`，`withContext(IO)` 切了线程为什么还错？"单线程多协程"一定安全吗？

### 机制级答案

- **withContext 不串行化**：它只保证**当前协程**切换执行线程；若两个协程都改 `count++`，即使都跑在**同一个单线程 Dispatcher** 上，也可能在挂起点交错——`count++` 若被拆成读-改-写且中间有挂起（或另一协程在无挂起间隙被切换——单线程调度通常在挂起点切换，但复合操作跨挂起仍不原子），结果错误。
- **复合操作原子性**：`count++` 本身无挂起时在单线程上不会被其他协程插入；**危险的是"读 → 挂起 → 写"模式**，或把 count 放进多线程 Dispatcher 的多个 launch。正确心智模型：**协程内无挂起点的临界段在单线程上安全；跨挂起或跨线程必须同步**。
- **多线程 Dispatcher**：Default/IO 上多个协程 = 真并行数据竞争，需要 Mutex/原子类/单线程隔离——`@Volatile` 只解决可见性不解决复合原子性。
- **不可变优先**：优先 `val` + 不可变集合 + 消息传递（Channel/Flow）缩小共享面；共享可变状态是最后手段。

### 【为什么这样设计】

协程调度在挂起点交错是 CPS 状态机的自然结果——语言不隐藏交错点，把并发控制交还给显式原语；与 Java 内存模型"共享可变需同步"同一哲学，只是执行单元从线程变为协程。

### 【面试官视角：考察点 / 加分 / 减分】

- 考察点：withContext 语义、单线程交错条件、@Volatile 边界、消息传递优先。
- 加分：能举"check-then-act"跨挂起例子；提到原子类在协程中仍可用但可能阻塞线程。
- 减分：说"单线程协程永远无竞争"；用 synchronized 包长挂起块（阻塞 Dispatcher 线程）。

### 【追问链】

1. 为什么不用 synchronized？→ 挂起函数不能在持锁时安全调用（持锁挂起易死锁/占线程）；应用 Mutex。
2. 原子类可以吗？→ 无挂起的简单计数可以；含 IO 的复合逻辑仍需 Mutex/单 actor。
3. 结构化并发能防数据竞争吗？→ 不能，只管生命周期与异常，不管共享变量。

---

## 二、Mutex vs Actor vs Channel

### 问题

三种原语分别适合什么场景？

### 机制级答案

- **Mutex**：挂起式互斥锁（`lock`/`unlock`/`withLock`）；等待者挂起不占线程；**不可重入**（同协程二次 lock 死锁）；临界区内尽量短、**避免在锁内做长 IO**（否则队头堵死后续）。
- **Channel**：并发安全的队列；发送/接收挂起；适合**生产-消费**、跨协程移交所有权；容量与溢出策略（第3页）决定背压。
- **Actor**（`actor {}` 构建器，演进中能力随版本；概念模型稳定）：单协程消费 Channel 消息、处理串行——**把共享状态收进一个拥有者**，外部只发消息；比"共享变量 + 锁"更易推理；失败策略要单独设计（子失败是否 supervisor）。
- **选型口诀**：互斥短临界 → Mutex；传递数据所有权 → Channel；共享状态机串行化 → Actor/单线程 dispatcher。
- **与 Java 对比**：Mutex ≈ 可挂起的轻量锁；Actor ≈ 单线程事件循环（Handler/looper 思想）但结构化在协程树内。

### 【为什么这样设计】

挂起锁避免线程阻塞；Channel/Actor 用"共享改为消息"消灭大部分锁需求——与 Erlang/Go CSP 一脉；Kotlin 在标准库提供最小原语，不强制单一模型。

### 【面试官视角：考察点 / 加分 / 减分】

- 考察点：Mutex 不可重入、Channel 所有权、Actor 串行状态。
- 加分：临界区持锁过长的队头阻塞；Mutex 与 Semaphore 区别（信号量许可数）。
- 减分：Actor 当万能吞一切异常；在 Mutex 内 delay 大段时间。

### 【追问链】

1. Mutex 和 lock.withLock 失败？→ 异常也 finally unlock；但锁内挂起取消时仍要保证解锁（withLock 处理）。
2. Channel 关闭语义？→ close 后接收在耗尽后结束；发送失败；用于结束协议。
3. 无 actor 构建器版本？→ 自建 `scope.launch { for (msg in channel) … }` 等价串行循环。

---

## 三、单线程多协程的正确共享

### 问题

在 Main/单线程上多个协程，如何安全共享？Android 里有什么现成模式？

### 机制级答案

- **安全来源**：同一 Dispatcher 单线程上，**无挂起的临界段**不会被其他协程插入——简单计数、更新非挂起数据结构可不加锁（仍要防未来换 Dispatcher）。
- **不安全**：临界段含挂起；或部分逻辑在 IO 后回写——用 Mutex 包住"检查-修改"，或把写操作全部 `send` 给单协程所有者。
- **Android 模式**：
  - UI 状态单一来源：ViewModel 内 StateFlow，唯一修改入口 `update { }`（原子更新函数，内部用 CAS/锁保护槽位，非业务长临界区）；
  - 事件经 Channel 由单收集者消费；
  - 避免"Fragment 与 Activity 同时改同一 VM 字段"——架构层消灭多写者。
- **update 算子**：`MutableStateFlow.update` 对**当前值槽**做原子 CAS 循环——适合"无挂起快速算新值"；算新值若含挂起，应先算完再 set，或换 Mutex。

### 【为什么这样设计】

单线程 + 挂起点协作模型给出"局部无锁"的可能性；update/Mutex/Actor 提供从简单到复杂的递进——正确性来自收敛写者，不是来自"碰巧单线程"。

### 【面试官视角：考察点 / 加分 / 减分】

- 考察点：挂起点交错、StateFlow.update 原子范围、多写者架构问题。
- 加分：提出"状态机 + 单写者"评审规则；测试中强制多协程压数据竞争。
- 减分：依赖 Main 单线程当全局锁；update 里调用 suspend。

### 【追问链】

1. update 里能 suspend？→ 不能当挂起临界区；API 为非挂起 lambda。
2. 多模块同时写 VM？→ code review/架构：唯一 repository 所有权。
3. 如何测出竞争？→ 多线程 TestDispatcher 或 Default 压力测试 + 不变式断言。

---

## 四、协程测试：TestDispatcher / 虚拟时间

### 问题

含 `delay`/IO 的协程单测如何做？`advanceUntilIdle` 在测什么？

### 机制级答案

- **问题**：真实 delay 让单测慢且不稳；真实 IO 依赖网络；多线程竞态难复现。
- **TestDispatcher 家族**（kotlinx-coroutines-test，语义级，API 名随版本微调）：
  - `runTest`：测试入口，**虚拟时间**——delay 不真等，推进虚拟时钟；
  - `StandardTestDispatcher`：任务入队，需 `advanceUntilIdle`/`runCurrent` 驱动；
  - `UnconfinedTestDispatcher`：立即执行到下一挂起点（更"急"，易掩盖顺序问题）；
  - 注入：被测代码接受 `CoroutineDispatcher`/`Main` 替换为 `Dispatchers.setMain(testDispatcher)`，避免生产代码硬编码 Default。
- **advanceUntilIdle**：运行调度队列直到空（虚拟时间全部到期）——断言"最终状态"；`advanceTimeBy` 测定时逻辑（防抖、超时）。
- **注意**：
  - 生产代码用 `viewModelScope`/注入 dispatcher，测试才能替换；
  - Flow 测试用 `Turbine` 类收集器（概念：挂起收集单个发射并断言）或手动 collect；
  - 真正的并发竞争测试仍要多线程压测，虚拟时间单线程不复现数据竞争；
  - `runTest` 有未完成协程/异常超时失败——泄漏会被抓。

### 【为什么这样设计】

虚拟时间把"并发逻辑正确性"与"真实时钟/真实线程"解耦——可重复、快速；注入 dispatcher 强迫依赖反转——可测性驱动架构，与界面测试注入调度器同一思想。

### 【面试官视角：考察点 / 加分 / 减分】

- 考察点：虚拟时间、注入而非硬编码、advanceUntilIdle 语义、与真并发测试边界。
- 加分：防抖/超时用例设计；StateFlow 用 test scope 收集断言；提到 CI 不跑真 delay 长等待。
- 减分：单测里 Thread.sleep；全局替换 Main 不恢复导致后续测试污染。

### 【追问链】

1. 真实 IO 怎么测？→ 抽象 Repository 接口 fake；或 MockWebServer 类边界。
2. runTest 超时？→ 默认有整体超时防挂；查未取消子协程。
3. 与 Java 并发测试关系？→ 压力测试、 jcstress 类工具思想可借鉴，但协程测试主路径是 TestDispatcher。
