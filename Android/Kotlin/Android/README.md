# Android 上的 Kotlin

## 一、生命周期感知协程

### 问题

`viewModelScope` / `lifecycleScope` 何时取消？`repeatOnLifecycle` 解决什么问题？

### 机制级答案

- **viewModelScope**：`ViewModel` 内置 `CoroutineScope`（SupervisorJob + Dispatchers.Main.immediate）；`onCleared` 时 cancel——**进程内配置变更（旋转）不清 VM，不取消**；适合与 VM 生命周期一致的加载与状态生产。泄漏面：VM 本身被长生命周期对象引用导致 onCleared 不来。
- **lifecycleScope**：Activity/Fragment 的 `LifecycleCoroutineScope`；`DESTROYED` 取消——适合页面级短任务；**旋转会取消**，结果不跨重建（需 VM 承接）。
- **collect 在 UI 的正确姿势**：`lifecycleScope.launch { repeatOnLifecycle(STARTED) { flow.collect { … } } }`——**STOPPED 停止收集**，避免后台更新 View/无意义计算；再次 STARTED **重新收集**（StateFlow 会再发当前值）。KTX 各版本有更短写法，语义相同。
- **陷阱**：
  - `lifecycleScope.launch { flow.collect }` 不加 repeatOnLifecycle → 后台仍收集，耗电/更新不可见 UI；
  - 在 `onCreate` launch 长任务而结果只写 View → 需结构判断用 VM；
  - Fragment viewLifecycleOwner 与 Fragment 自身 scope 混用 → view 已销毁仍写 binding。

### 【为什么这样设计】

作用域与生命周期对象对齐是结构化并发在 Android 的落地；repeatOnLifecycle 把"订阅节奏"从"创建到销毁"细化为"可见性节奏"——订阅者生命周期应匹配数据消费者真正需要数据的时段。

### 【面试官视角：考察点 / 加分 / 减分】

- 考察点：onCleared vs DESTROYED、旋转行为、STARTED 收集、viewLifecycleOwner。
- 加分：多说清"VM 跨旋转、View 不跨"的数据分工；提到 process death 与 SavedState。
- 减分：全部 GlobalScope 或 GlobalScope 替代 VM；在 onDestroy 后还手动回调 View。

### 【追问链】

1. STARTED 与 RESUMED 差别？→ 分片动画等场景用 RESUMED；默认 UI 更新 STARTED 足够。
2. process death？→ VM 也死；持久层 + SavedStateHandle（见下）。
3. 服务端推送更新 UI？→ 经 VM 状态合并，不在推送回调直接碰 View。

---

## 二、StateFlow / SharedFlow 替代 LiveData

### 问题

何时用 StateFlow/SharedFlow，何时保留 LiveData？有哪些迁移坑？

### 机制级答案

- **对比**：

| 维度 | LiveData | StateFlow | SharedFlow |
|------|----------|-----------|------------|
| 生命周期感知 | 自动（观察者） | 需 repeatOnLifecycle 等 | 同左 |
| 线程 | 主线程规范 | 任意线程读写，收集线程由 scope 决 | 同左 |
| 初始值/当前值 | 可有 | 必有 initial | 可选 replay |
| 粘性 | 有 | 有（当前值） | 按 replay |
| 操作符/测试 | 弱 | 强（Flow） | 强 |
| 多平台 | AndroidX | 协程生态 | 协程生态 |

- **选型**：新代码/需操作符/跨平台 → Flow；存量简单 UI 绑定可留 LiveData；**不要在无生命周期收集的裸 launch 里收 StateFlow 更新 View**。
- **迁移坑**：
  - LiveData 的 `setValue` 主线程约束消失后，后台线程写 `state.value` 合法——注意业务仍要单写者；
  - 事件类用 StateFlow → 旋转/重订阅重放 toast → 改 Channel 或消费幂等；
  - 相同值 `equals` 不通知 → UI 依赖"赋值即刷新"失效；
  - 初始值空列表闪一下 → 用 sealed Loading/Content 或占位策略。

### 【为什么这样设计】

Flow 是通用响应式底座，LiveData 是 Android 生命周期特化容器——二者可互操作（`flow.asLiveData()` / `liveData.asFlow()`）；迁移成本主要在**收集侧生命周期责任**从框架转到调用方。

### 【面试官视角：考察点 / 加分 / 减分】

- 考察点：粘性、收集生命周期、事件 vs 状态、equals 去重。
- 加分：给出模块级迁移策略（先 VM 状态后 UI）；提到 Turbine/TestDispatcher 测试优势。
- 减分：说 LiveData 一无是处；StateFlow 当事件总线。

### 【追问链】

1. SharedFlow replay 取多少？→ 状态类可不用 SharedFlow；事件 0；最近 N 条恢复按产品。
2. 与 Compose collectAsState？→ 在组合期收集，仍要后台策略与 lifecycle 意识。
3. 混用两套？→ 边界处转换可以，长期双源状态必出不一致。

---

## 三、Kotlin 属性委托与 ViewModel SavedState

### 问题

`SavedStateHandle` 如何与委托结合？哪些数据不该进 SavedState？

### 机制级答案

- **模式**：`val name: MutableLiveData<String> = savedStateHandle.getLiveData(KEY)` 或 StateFlow 版 `savedStateHandle.getStateFlow(KEY, "")`——**单一来源**，恢复进程死亡后的可 parcel 数据；VM 复用同一 handle。
- **委托形态**：AndroidX 提供 `savedStateViewModel` / `viewModels` / `SavedStateHandle.saveStateProvider` 等——本质是属性委托 + 工厂创建；注意 **delegate 只在首次访问创建**，与 `by viewModels` 缓存语义一致。
- **装什么**：用户输入草稿、导航参数、轻量 UI 状态（分页 token、选中 id）；**不装**：大 bitmap、数据库实体全量、可从持久层重推的数据、会过期的会话权威数据（应放 Repository）。
- **进程死亡恢复**：SavedState 是 Bundle 路径，类型要可序列化；恢复失败/缺 key 用初始值兜底；与配置变更恢复区分（后者内存对象还在）。

### 【为什么这样设计】

SavedStateHandle 是"系统可保存小状态"与"VM 生命周期"的接缝——委托让样板消失；容量与类型限制来自 Bundle 事务开销与 binder 约束，塞大对象会卡保存/恢复甚至 TransactionTooLarge。

### 【面试官视角：考察点 / 加分 / 减分】

- 考察点：process death vs 旋转、getStateFlow 单一来源、Bundle 容量。
- 加分：表单草稿自动保存产品体验；恢复版本兼容（key 演进）。
- 减分：把整个登录态 user 塞 SavedState 当唯一持久化；不说 process death。

### 【追问链】

1. SavedState 和 DataStore？→ 权威持久用 DataStore/DB；SavedState 只保"小而急"。
2. 自定义类？→ 需 Parcelable/Serializable；空安全与版本字段。
3. 多 VM 共享 handle？→ 按 owner 作用域；Fragment view 与 activity 级区分。

---

## 一、Kotlin 与 View 体系 / Compose 互操作

### 问题

View 体系与 Compose 混用时，Kotlin 语法与空安全有哪些注意点？

### 机制级答案

- **ViewBinding**：生成非空字段（视图存在性由绑定生命周期保证）；`binding.root` 布局为 null 的旧问题随 viewBinding 稳定减弱；Fragment 中在 `onDestroyView` 置空 binding 是防泄漏模板（可空 var + 委托模式）。
- **findViewById**：返回平台可空，需 `?.` 或约定非空——优先 ViewBinding。
- **Compose 互操作**：
  - `composeView.setViewCompositionStrategy` 控制 dispose 时机，防泄漏；
  - View 中嵌 Compose：`ComposeView.setContent`；Compose 中嵌 View：`AndroidView`；
  - 状态桥接：`mutableStateOf` 与 ViewModel 的 StateFlow 边界——UI 用 `collectAsState`，不要把 `MutableState` 暴露给 VM；
  - Kotlin lambda/`@Composable` 重组：理解"读状态才订阅"——误在组合中读无关大对象浪费；稳定类型与不可变有助于跳过重组（语义级）。
- **SAM 与监听**：`view.setOnClickListener { }` Kotlin lambda；老接口需 `object :`。

### 【为什么这样设计】

Compose 声明式 UI 与 Kotlin 语法天然亲和；与 View 互操作层刻意保守以渐进迁移——坑主要在**生命周期 dispose** 与**状态所有权**，不是语法。

### 【面试官视角：考察点 / 加分 / 减分】
- 考察点：binding 置空、composition strategy、状态单向数据流。
- 加分：重组与稳定性的关系一句话；混排页面的生命周期一致性。
- 减分：在 Compose 里直接改 ViewModel 可变字段；泄漏 composition。

### 【追问链】

1. 何时强制单 Compose？→ 新页面全 Compose，混排设边界与路由。
2. Recycler + Compose item？→ 逐条 ComposeView 或 LazyColumn 整页化，按性能取舍。
3. 空安全差异？→ Compose 参数非空默认；View 系统平台类型仍在。

---

## 二、空安全在 Parcelable / 序列化边界

### 问题

Parcelable、JSON、Bundle 与 Kotlin 可空类型如何配合？

### 机制级答案

- **Parcelable**：`@Parcelize`（kotlin-parcelize）生成 writeToParcel/CREATOR；**Java 反射式实现与手写 CREATOR 在 null 上易错**——Parcelize 按类型处理可空；未实现 Parcelable 的类型不能直接塞。
- **跨进程契约**：对端是 Java/旧版本时，Bundle 读出是平台类型——`bundle.getString` 可空；约定字段必须 `?:` 或 `requireNotNull` 收敛。
- **JSON**：反射序列化（Gson 等）对 Kotlin 非空字段传 null 可能在构造期或 setter 摆异常/写入坏值——用支持 Kotlin 元数据的库（Moshi/Kotlinx.serialization，版本能力语义级）并在边界校验；`data class` 默认值与缺失字段行为要测。
- **Kotlinx.serialization**：编译器插件生成序列化器，保留可空与默认值语义——边界首选；多态 sealed 需注册子类。
- **红线**：任何**反序列化对象**当非空用之前视为不可信——远程/磁盘数据是空安全与类型安全的外部边界，校验在边界做。

### 【为什么这样设计】

Parcel/JSON 是进程与语言边界，类型信息易丢失；@Parcelize/序列化插件把契约写进生成代码，减少手写 null 分支错误。

### 【面试官视角：考察点 / 加分 / 减分】
- 考察点：Parcelize 与手写差异、getString 可空、不可信边界。
- 加分：版本化 schema、缺字段默认值策略；崩溃与脏数据关联案例。
- 减分：反序列化直接 `!!`；不测 process death 恢复。

### 【追问链】

1. TransactionTooLarge？→ Binder 缓冲限制；大图用磁盘 URI。
2. 枚举/密封序列化？→ 多态注册与未知值兼容策略。
3. 混合语言模块？→ 以边界 DTO 注解/校验为准。

---

## 三、列表路径：内联与装箱意识

### 问题

RecyclerView/大列表热路径上，Kotlin 有哪些隐性成本要意识？

### 机制级答案

- **每帧分配**：onBind 里高阶链、正则、data class 临时对象增加 GC 压力——热路径重用 DiffUtil 计算之外的对象复用/不可变共享。
- **装箱**：`List<Int>`/`SparseArray` 场景；Int 进泛型装箱；用原始类型数组/专用容器敏感路径。
- **内联集合操作**：`items.map{}.filter{}` 若每次 bind 链很长且列表极大，考虑预处理或 Sequence 短路（第 Language/3 页）。
- **协程**：在 bind 中 launch 是反模式；滚动时用列表项 job 取消/图片库生命周期（系统设计页）而非裸协程写 View。
- **Compose Lazy 列表**：key 稳定、contentType、不可变 state 列表避免全量重组。

### 【为什么这样设计】

UI 帧预算紧，Kotlin 抽象的分配与虚调用在热路径被放大——优化前提是**先测量**（Profiler 分配/GC），避免凭感觉全盘去抽象。

### 【面试官视角：考察点 / 加分 / 减分】
- 考察点：bind 中分配、装箱、协程误用。
- 加分：以分配率/GC 时间数据说话；DiffUtil 与不可变 item 模型。
- 减分：见 map 就改 for 无测量；在 bind 用 GlobalScope。

### 【追问链】

1. 如何测？→ Android Studio Profiler、分配追踪、宏观帧掉帧指标。
2. Bitmap 之外的大对象？→ 复用池、分页加载。
3. Flow 在列表？→ 列表数据源冷流在 VM 收，UI 只收 State。

---

## 四、编译产物：metadata 与默认参数（一句话深度）

### 问题

Kotlin metadata、默认参数合成对包体积和 Java 互操作意味着什么？

### 机制级答案

- **kotlin metadata**：类注解携带 Kotlin 类型信息，供 Kotlin 编译器/反射/序列化插件使用——增加少量注解体积；混淆时保留规则要覆盖（R8/ProGuard 默认有 Kotlin 规则模板）。
- **默认参数**：`fun f(a: Int, b: Int = 0)` 合成 `$default` 合成函数 + 调用点分发；大量公开默认参数 API 在 Java 侧需 @JvmOverloads 重载，**方法数与 dex 体积随重载组合增长**。
- **inline**：调用点膨胀与方法内联的包体积权衡（Language/2）；慎对极热库函数无节制 inline 公共 API。
- **R8**：Kotlin 属性/合成类可被 shrink；未用的委托/lazy 减少；正确 keep 才能保住反射序列化——包体积治理与 Kotlin 特性评审一体。
- **一句话**：metadata 几乎不影响用户感知体积；默认参数与 inline 的**调用点膨胀**才是 dex 敏感点——库作者控制 API 面，应用作者看 shrink 后指标。

### 【为什么这样设计】

metadata 换来跨编译单元的类型精度；默认参数避免重载爆炸但用调用点复杂度换——两套机制都在"语言体验"与"产物大小/互操作"之间取舍。

### 【面试官视角：考察点 / 加分 / 减分】
- 考察点：metadata 用途、$default、与 R8 关系。
- 加分：提到 binary-compatibility 与 api 校验；inline 对 dex 的实测意识。
- 减分：声称 metadata 是主要包体积元凶；乱 keep 全规则导致包更大。

### 【追问链】

1. 混淆后反射序列化挂了？→ keep 元数据与序列化插件生成类。
2. 默认参数跨模块二进制兼容？→ 默认值变更可能影响调用点合成——见 Senior/2。
3. 如何看方法数？→ dex count 工具与 shrink 报告。
