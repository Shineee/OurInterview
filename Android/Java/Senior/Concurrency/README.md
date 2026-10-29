# Concurrency

## 一、AQS（AbstractQueuedSynchronizer）源码级问答

### 问题

AQS 是 J.U.C 的骨架，请讲清它的核心结构与一次 `lock()/unlock()` 的完整路径。

### 机制级答案

- **核心结构**：
  - `state`（volatile int）：同步状态。子类用它表达不同语义——ReentrantLock：0=未锁，值=重入次数；Semaphore：剩余许可；CountDownLatch：剩余计数；ReadWriteLock：高低位切分读/写状态。
  - **CLH 变体双向队列**：失败的线程封装为 `Node`（含 waitStatus、prev/next、thread、模式标志）进入 FIFO 等待队列；AQS 用自旋 + `LockSupport.park/unpark` 做阻塞唤醒，而不是纯自旋烧 CPU。
  - **Condition 队列**：每个 ConditionObject 维护独立单向等待队列；`await` 把节点从同步队列转移到 Condition 队列并 park，`signal` 再搬回同步队列尾部参与锁竞争——所以 `signal` 之后到真正拿锁之间仍有延迟。
- **独占路径（ReentrantLock.lock）要点**：
  1. `tryAcquire`（子类）：CAS 改 state；`fair` 先看队列是否有等待者，非公平直接 CAS 插队——非公平吞吐高的原因是省掉了部分 park/unpark 往返；
  2. 失败 → 入队（`addWaiter`，`enq` 保证队列初始化的 CAS 停等安全）→ `acquireQueued`：自旋中前驱是 head 时再试 `tryAcquire`，失败则 `shouldParkAfterFailedAcquire`（处理 cancel/信号态）后 `parkAndCheckInterrupt` 挂起；
  3. `unlock` → `tryRelease`：state 归零后 `unparkSuccessor` 唤醒后继。
- **共享路径（Semaphore/CountDownLatch）**：`tryAcquireShared` / `releaseShared` 返回值约定负失败；传播式唤醒（`doReleaseShared`）把信号沿队列传下去——读写锁的读共享与 CountDown 的多等待者复用此路径。
- **可中断/超时/条件锁**：`lockInterruptibly` 与普通 `lock` 的差异在入队前检查与 `acquireInterruptibly` 的中断响应；`tryLock(timeout)` 走 `doAcquireNanos`。
- **实现者视角**：自定义同步器只需正确实现 `tryAcquire/tryRelease`（或 shared 变体），队列与唤醒由 AQS 保证——模板方法模式在并发框架中的教科书应用。

### 【为什么这样设计】

- 状态与队列分离：状态语义留给子类，排队/唤醒等难写对的基础设施复用一份——降低 J.U.C 出错面。
- CAS 自旋短临界 + park 长等待：在多核上兼顾无锁快路径与低 CPU 占用慢路径。
- 非公平默认：吞吐优先的工程取舍，公平锁留给有明确秩序需求的场景（其代价是吞吐与上下文切换）。

### 【面试官视角：考察点 / 加分 / 减分】

- 考察点：state 语义映射、队列节点状态流转、Condition 的双队列、锁公平性。
- 加分：能讲 `head 虚拟节点`、`cancel 节点的摘除`、`unpark 后中断与重试循环`；用 AQS 结构解释 `ReentrantReadWriteLock` 降级（写锁降级为读锁需在持有写锁时获取读锁）。
- 减分：只背"CLH 队列"名词；说不清 `lock` 失败后到底 park 在哪里；把 Condition 当作 Object.wait 的简单包装而不提队列转移。

### 【追问链】

1. 为什么 AQS 用双向队列而 CLH 原版是单向？→ 取消/超时节点需要快速断开前驱关系，双向便于跳过取消节点。
2. `tryAcquire` 里要不要检查中断？→ 入口检查与否区分 `lock` 与 `lockInterruptibly`。
3. AQS 与 StampedLock/LongAdder 思路差异？→ 后者用分段/乐观读避开 AQS 队列，适合读多写极少的计数场景。

---

## 二、线程池设计与调优（场景题）

### 问题

线上服务用线程池处理任务，出现"任务堆积、RT 升高、偶发拒绝"。如何设计与调优一个线程池？

### 机制级答案

- **参数与执行语义（`ThreadPoolExecutor`）**：提交任务 → 线程数 < corePoolSize 直接建线程执行；否则入队；队列满才建线程到 maximumPoolSize；再超走拒绝策略。**注意**：在无界队列下 maximumPoolSize 形同虚设——这是"越调越大仍堆积"的经典根因。
- **队列选择**：
  - `LinkedBlockingQueue` 无界/大界：隔离突发，但无背压 → 内存与延迟失控；
  - `SynchronousQueue`/`ArrayBlockingQueue` 有界：强制背压 + 拒绝信号，适合对延迟敏感的服务；
  - 优先级/延迟队列：有明确 QoS 时才用，警惕饿死。
- **拒绝策略**：Abort（快速失败，默认合理）、CallerRuns（背压回提交线程，注意别在事件循环线程里 CallerRuns）、Discard/DiscardOldest（仅在"旧任务可丢"业务下用）。拒绝要接监控告警，不能静默。
- **参数经验值（场景化表达，不背死数）**：
  - CPU 密集：线程数 ≈ CPU 核数（+1 量级用于缺页等间隙）；
  - IO 密集：线程数 ≈ 核数 × (1 + 等待时间/计算时间)，用压测标定；上限受下游容量与连接池约束——**线程池大小本质是下游资源预算问题**；
  - 阿里线程池规范的精神：不同业务隔离多个池 + 有界队列 + 明确拒绝，防一个业务拖垮全局（资源隔离思想）。
- **调优作业流**：
  1. 定位堆积点：队列深度、活跃线程、任务提交速率与完成速率（打点）；
  2. 区分"慢在执行"（RT 高）还是"慢在排队"（执行快但队列长）——前者优化任务本身/扩容，后者调队列与并发；
  3. 看下游依赖（DB 连接池、HTTP 连接池）是否成为真瓶颈，盲目加线程只会放大下游雪崩；
  4. 灰度验证：拒绝率、P99、下游错误率三线一起看。
- **关闭与泄漏**：`shutdown` + `awaitTermination` 超时后 `shutdownNow`；任务里阻塞调用要有超时，否则池子关不掉；线程工厂命名与 UncaughtExceptionHandler 必配（与 Android 端线程池治理同构）。

### 【为什么这样设计】

线程池的三段式（core→队列→max→拒绝）本质是**在"快速拒绝"与"吸收突发"之间做缓冲管理**；工程上错误的队列无界化会让整套背压机制失效。隔离多个池是故障域思维：并发资源也要做爆炸半径控制。

### 【面试官视角：考察点 / 加分 / 减分】

- 考察点：三段式真实语义（很多人背错顺序）、无界队列陷阱、下游容量思维、监控字段。
- 加分：给出 `ThreadPoolExecutor` 监控项（active、poolSize、queueSize、completed、rejected）的告警阈值思路；提到 ForkJoinPool 的 work-stealing 适用场景（任务可分裂、CPU 密集）与 `ManagedBlocker`。
- 减分：公式党只背核数×2；建议无界队列"反正内存大"；忽略拒绝监控。

### 【追问链】

1. `submit` 与 `execute` 的差异？→ submit 包 FutureTask，异常被吞进 Future，不配 `get` 会静默丢失——线上故障常见根因。
2. 为什么核心线程也会超时？→ `allowCoreThreadTimeOut` 可开；弹性池在流量低谷释放资源。
3. 与虚拟线程如何取舍？→ 见 JVMEngineering 第2页：IO 密集的海量并发优先虚拟线程，池的职责转向"资源与并发度闸门"。

---

## 一、Disruptor 与无锁队列概念

### 问题

为什么 Disruptor 能比 `BlockingQueue` 快？讲讲环形缓冲与它依赖的内存模型知识。

### 机制级答案

- **结构**：预分配大容量数组（长度取 2 的幂，用位与取模），生产者写 `cursor` 递增位置，消费者跟踪自己的 sequence；**数组槽复用**而非节点分配——消灭队列节点的 GC 压力与 cache miss 来源之一。
- **等待策略（无锁协同核心）**：
  - 生产者超前消费者一圈时不能覆盖未消费数据——通过 ` gating sequence`（消费者序列的最小值，常配 `SequenceBarrier`/多消费者的依赖图）限制生产者；
  - 消费者落后时自旋 + `Thread.yield`/`LockSupport.park` 分级等待；策略可配（busy spin / yielding / blocking），本质是延迟 vs CPU 的 trade-off。
- **内存屏障与可见性**：序列字段用 volatile / VarHandle 语义发布，保证「先写数据、后写 sequence，读方先读 sequence 再读数据」的顺序与可见性——这是 JMM happens-before 在环形缓冲上的直接应用；不加屏障的"无锁"是数据竞争 UB，面试要能一眼戳穿。
- **伪共享（false sharing）**：相邻的生产者/消费者序列字段在同一 cache line 上时，各自写入使对方 cache line 失效，多核下性能雪崩。对策：`@Contended`（需 JVM 旗标）或手动 padding 到 cache line（64B 量级）。
- **批量/多生产者**：多生产者用 CAS 抢位置 + 填充槽位的空洞标记（占位），比单消费者单生产者复杂一个量级——面试讲清"最快路径是 SPSC"即可。
- **与 J.U.C 的对比叙事**：`ArrayBlockingQueue` 用一把锁+两个条件队列；Disruptor 把互斥变成 CAS + 缓存友好的环——适合**低延迟、高吞吐、任务可丢弃策略清晰**的场景（交易、日志、管道）；一般业务服务的瓶颈在下游 IO，上 Disruptor 是过度设计。

### 【为什么这样设计】

- 队列的性能敌人：分配与 GC、锁竞争、cache line 争用、多余屏障。Disruptor 把四项全部显式工程化——它是"用 JMM 知识换延迟"的范本，而不是魔法。
- 面试价值不在背类名，而在于证明候选人**能把内存模型落到数据结构设计**。

### 【面试官视角：考察点 / 加分 / 减分】

- 考察点：环形缓冲复用、gating sequence、伪共享、屏障顺序。
- 加分：说清 SPSC 与 MPMC 复杂度差异；提到 RingBuffer 与事件工厂预分配对象池。
- 减分：只说"无锁所以快"；不知道伪共享；建议所有队列都换 Disruptor。

### 【追问链】

1. 消费者崩溃不推进 sequence 会怎样？→ 生产者被 gating 阻塞——需要超时跳过或监控告警的业务语义。
2. 与 `ConcurrentLinkedQueue` 比？→ CLQ 是无界链表 CAS（Michael-Scott 思路），GC 与缓存局部性差于环形数组；延迟敏感场景 Disruptor 通常更稳。

---

## 二、并发 bug 排查（死锁 / 活锁 / 可见性）

### 问题

线上进程 CPU 飙高或线程全体"假死"，如何排查？

### 机制级答案

**1. 死锁（deadlock）**

- 定义：循环等待 + 互斥 + 不可剥夺 + 持有并等待——四条件同时成立。
- 排查：`jstack <pid>`（或 `jcmd <pid> Thread.print`）自带 deadlock 检测，直接给出环与持有链；Android 侧对应 ANR 归因中的锁等待桶（Stability 第2页）。
- 预防：全局锁序（按对象 hash/固定 id 加锁）、缩小锁范围、用 `tryLock` 超时破环。

**2. 活锁（livelock）与自旋风暴**

- 定义：线程都在动但互相让路/重试，无进展（如无退避的 `tryLock` 失败重试、双线程互相礼让）；CPU 高、无死锁报告。
- 排查：线程栈反复出现同一组 tryAcquire/回滚模式；结合 CPU 采样（perf/async-profiler）看热点是否在锁重试路径。
- 预防：随机退避（backoff + jitter）、避免在重试循环里持有其他锁。

**3. 可见性与指令重排 bug**

- 典型模式：无 volatile 的停止标志位循环不退出；双重检查锁（DCL）单例不加 `volatile`——半初始化对象被其他线程看见（重排 + 可见性双重问题）；生产者-消费者只用普通数组不发布。
- 排查难点：栈不"卡住"，逻辑偶发错误——靠压力测试 + 断言不变量；代码审计搜"跨线程共享的可变字段无同步"。
- 修复原则：**每个共享可变状态必须回答三个问题**——谁写、谁读、用什么同步（synchronized/volatile/原子类/不可变）。答不出就是隐患。

**4. Happens-before 速查框架（面试表述工具）**

- 程序序内（单线程 as-if-serial 保证语义）；
- unlock→后续 lock 同把锁；
- volatile 写→后续 volatile 读；
- 线程 start/join；

  - 传递性：A hb B、B hb C ⇒ A hb C。
- 公式用法：排查可见性 bug 时，在"写"与"读"之间找一条 hb 链；找不到就要补同步。

**5. 作业流（给面试官的 SOP）**

```
症状分类（假死 / CPU高 / 结果错）
  → jstack 多次采样（间隔1-2s）对比：卡同一处=阻塞/死锁；漂移=livelock/正常忙碌
  → 有无 DEADLOCK 报告；无则看锁持有（monitor / ownable synchronizer）
  → CPU 高：top -H + 线程 id 转 hex 对齐线程名，perf 采样热点
  → 结果错：审计共享状态同步；jmm 模式匹配（volatile/DCL/发布安全）
  → 复现优先：JCStress 类工具验证疑似 JMM 问题（概念级）
```

### 【为什么这样设计】

并发 bug 的共同点是**时间维上的错误**：静态代码审查与单线程测试无法覆盖，必须用"运行时证据链"（多帧线程栈、CPU 栈、JMM 模式）重建时间线——与 Android 生死簿思想同构：错误现场不可复现时，留下可交叉验证的痕迹。

### 【面试官视角：考察点 / 加分 / 减分】

- 考察点：四条件死锁模型、jstack 工具链、DCL 与 volatile、hb 链推理。
- 加分：主动区分 livelock 与 busy-wait；提到 `LongAdder`/`ConcurrentHashMap` 段思想作为"减少锁面"的正面案例；给出多次采样的操作细节（单次 jstack 会撒谎）。
- 减分：死锁=加锁顺序背诵无工具路径；把 `synchronized` 与 `volatile` 职责混淆（互斥 vs 可见性）。

### 【追问链】

1. `ReentrantReadWriteLock` 为什么写锁饥饿而读锁可能饥饿？→ 写写互斥、写读互斥；读多写少时写者等待，非公平策略下可能久等——StampedLock 乐观读缓解读侧开销。
2. 原子类为什么有时比锁快？→ CAS 无阻塞快路径；高争用下自旋失败率高反而慢（LongAdder 分段缓解）——"无锁一定快"是错的。
