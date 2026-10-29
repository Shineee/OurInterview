# Architecture 面试题

* MVC / MVP / MVVM 的本质差异与演进动机（不是背名词，而是讲清"解决了什么问题、引入了什么新问题"）

  > **MVC**：Activity 同时是 Controller + View 初始化者，Model 独立。问题：Activity 膨胀为"上帝对象"，逻辑与视图难单测，生命周期感知差（旋转丢状态）。
  >
  > **MVP**：Presenter 持有 View 接口，View（Activity/Fragment）变哑（只实现接口），Presenter 可脱离 Android 类单测。问题：接口样板爆炸（每个操作一个方法），Presenter 生命周期需手动管理，容易持有 View 引用泄漏。
  >
  > **MVVM**：View ↔ ViewModel（不持 View 引用，暴露可观察数据）← Model。DataBinding/LiveData/StateFlow 让数据流向单向；ViewModel 在配置变更中存活，天然解决旋转丢数据；View 可订阅无需 Presenter 引用。新问题：数据流调试复杂、滥用 observable 导致"数据流意大利面"、绑定表达式调试难。
  >
  > 演进主线：**把"状态与生命周期"从 View 中剥离，并让数据流可观察、可单测**。面试讲到"为什么 MVVM 胜出：生命周期安全 + 可测试 + 单向数据流"即到位。

* Jetpack Lifecycle 的实现原理（源码级：为什么能感知生命周期）

  > 核心：**LifecycleObserver + LifecycleRegistry**。每个 LifecycleOwner（Activity/Fragment）持 LifecycleRegistry；系统回调（Activity 的 reportFragment / Fragment 的事件分发）转成 Lifecycle.Event（ON_CREATE/ON_START/ON_RESUME...）driven 到 registry。
  >
  > 实现关键：**Generated adapter 或 默认 lifecycle-event 回调反射**；Jetpack 推荐编译期生成 LifecycleObserver adapter 避免反射。ComponentActivity 构造时注册 ReportFragment 无 UI 分身来分发事件。
  >
  > 好处：任意组件（非 View）可订阅生命周期（如相机、传感器、协程 scope 自动取消）。这正是 LiveData/生命周期感知协程的基石。

* ViewModel 为什么能在配置变更中存活？源码里发生了什么

  > Activity destroy（配置变更）时，**ViewModelStore 不销毁**（存于 NonConfigurationInstances，随 ActivityClientRecord 传递）；新 Activity attach 时取回同一 ViewModelStore，ViewModel 实例复用（onRetainNonConfigurationInstance 机制）。
  >
  > **进程死亡**：整个 NonConfigurationInstances 没了 → ViewModel 清空；要恢复需 **SavedStateHandle**（内部 Bundle 走 onSaveInstanceState 链路，系统进程保管）。
  >
  > 面试区分度：能讲"配置变更保留（ViewModelStore）vs 进程死亡恢复（SavedStateHandle/onSaveInstanceState）两套机制"，直接对齐 Activity 状态保存深度。

* LiveData 与 StateFlow/SharedFlow 怎么选（含粘性事件问题）

  > **LiveData**：生命周期感知（STARTED 以上才收，避免泄漏），粘性（新订阅者立刻收到最后值），setValue/postValue 简单。局限：单一值、操作符少、与 Kotlin 协程生态略隔。
  >
  > **StateFlow**：总有初值、热流、不粘历史（replay=1 即最后值语义类似）、支持协程操作符。需 **repeatOnLifecycle** 收集，否则页面后台仍收（浪费/错逻辑）。
  >
  > **SharedFlow**：可配置 replay/buffer/extraBufferCapacity，适合事件流（replay=0 防"新订阅者吃到旧事件"的粘性坑；需要重放用 replay=1）。
  >
  > 选型：新项目 Kotlin + 协程 → StateFlow/SharedFlow；存量或团队偏 Java → LiveData。关键是理解**粘性**带来的"旋转后旧事件重放"bug——这是面试高频追问。

* Room 的编译期校验与 @Dao 生成，和直接用 SQLite 的区别

  > Room 在**编译期**对 @Query SQL 做语法/返回类型校验（错误编不过，而非运行时崩），自动生成 DAO 实现（非反射，性能好）；支持 LiveData/Flow 返回、迁移 Migration、@Transaction。
  >
  > 相比裸 SQLiteOpenHelper：类型安全、可观察查询、编译期挡 SQL 错、迁移友好；代价是多一层生成与学习成本。
  >
  > 追问：Room 的线程——默认不允许主线程查询（allowMainThreadQueries 是反模式）；挂起 DAO 函数自动切 IO。迁移忘写 fallbackToDestructiveMigration 会崩——线上升级注意。

* DataStore 与 SharedPreferences 的对比与选型

  > **SharedPreferences**：apply 异步写内存立即生效+排队写盘；commit 同步阻塞。问题：首次 get 阻塞读盘、ANR 风险、多进程不安全（MODE_MULTI_PROCESS 已不可靠）、类型弱（字符串堆砌）。
  >
  > **DataStore**：Jetpack 替代，基于 Proto（强类型）或 Preferences（键值），**Flow 暴露、协程写入、事务性更新、无 ANR**（IO 线程）。不支持跨进程（官方说单进程；跨进程需自管或回退 DB）。
  >
  > 选型：新代码默认 DataStore；存量 SP 按需迁；多进程配置不要用 SP 多进程模式，改 ContentProvider/DB。

* 面试官视角

  > **考察点**：架构演进的因果逻辑、Jetpack 组件底层机制（而非只会 API 调用）。
  >
  > **加分**：Lifecycle 的 ReportFragment 分发、ViewModel 的 NonConfigurationInstances、StateFlow 粘性坑、Room 编译期校验。
  >
  > **减分**：只会背"MVVM 解耦"；以为 ViewModel 进程死也能活；推荐 SP 做多进程配置。

---

* 为什么需要依赖注入？Dagger/Hilt 解决什么问题、核心概念（Component/Scope/Qualifier/Module）

  > **DI 本质**：把"对象创建与依赖装配"从使用方剥离，由框架/容器注入，解耦构造、便于替换 mock、管理生命周期。
  >
  > 手写 DI（Service Locator/构造传递）在小项目够用；大项目依赖图复杂时手写爆炸。
  >
  > **Dagger2 核心**：
  > - @Inject：标注构造注入或字段注入点。
  > - @Module + @Provides/@Binds：提供无法构造的依赖（接口实现、三方类）。
  > - @Component：依赖图容器，连接 Provider 与 Inject 点；@Singleton/@ActivityRetainedScoped 等 Scope 保证单例生命周期与图作用域一致（不同 Scope 不能混用错会编译错）。
  > - @Qualifier：区分同类型不同依赖（@BaseUrl / @AuthOkHttp）。
  >
  > **Hilt**：Dagger 的官方上层封装，用 @HiltAndroidApp、@AndroidEntryPoint、@HiltViewModel 等预定义 Component 层级（Singleton/ViewModel/Activity/Fragment），**减少模板**，编译期生成同 Dagger 代码。选型：新项目直接 Hilt；深度定制图才裸 Dagger；Koin 等运行时 DI 胜在轻但无编译期校验。
  >
  > 追问："DI 在 Android 的价值？"——ViewModel 作用域绑定、多绑定 Set（多实现注册）、测试替换。

* 模块化/组件化：分层边界怎么划、依赖如何治理

  > **分层**：
  > - app（壳工程，只做组装与主题）。
  > - feature/*（业务组件：home、profile、pay...相互不依赖）。
  > - core/*（公共能力：network、database、ui-kit、router、analytics）。
  > - lib/*（纯工具，无业务）。
  >
  > **依赖规则**：单向、禁止 feature 互相依赖（跨 feature 通信走 **路由/接口下沉**）；core 不依赖 feature；用 Gradle 的 api/implementation 控制传递依赖（implementation 隐藏内部细节避免泄漏）。
  >
  > **治理手段**：
  > - 依赖锁（version catalog / lockfile）统一三方版本。
  > - ArchUnit/Lint 依赖规则检查（禁止 feature→feature import）。
  > - 接口下沉（SPI）：把跨模块契约放 core-api 模块，实现放 feature。
  >
  > 面试考察：能否说出"为什么禁止横向依赖"（耦合回归单体、编译依赖爆炸、无法独立发版）而非只会"分模块更清晰"。

* 组件间通信方案对比与选型（路由、接口、事件总线、DeepLink）

  > - **路由（ARouter 类比/自研）**：字符串/类路径跳转 + 目标 Service 接口（IRouteService）解耦。适合页面跳转与跨模块方法调用；要处理降级（目标不存在）。
  > - **接口下沉 + Provider**：编译期依赖 core-api，运行时注册实现（SPI/ServiceLoader/Hilt Multibinding）。适合稳定契约，类型安全优于字符串路由。
  > - **事件总线（SharedFlow/EventBus）**：广播式，弱耦合但时序与粘性难控，慎作主通信通道。
  > - **DeepLink**：对外/跨 App 入口，内部也可统一，但需中心化管理防冲突。
  >
  > 大型 App 常见组合：页面跳转用路由，能力调用用下沉接口+DI 多绑定，状态广播用 SharedFlow（受控）。

* 单一 Activity（Navigation）与多 Activity 的取舍

  > **单 Activity + Navigation/Compose**：转场统一、共享元素自然、模块化清晰、避免 task 栈混乱；代价是"返回栈语义"全在 Navigation 内部，深度链接与系统返回要适配 Predictive Back；大页面内存集中于一个 Activity。
  >
  > **多 Activity**：天然 Task 管理、系统级返回/分屏支持好；代价是跨 Activity 状态共享复杂、转场与主题切换碎片化。
  >
  > 现实：新项目/Compose 偏单 Activity；存量大 App 常"业务内部单 Activity，跨业务多 Activity + 路由"。面试讲取舍优于站队。

* 面试官视角

  > **考察点**：DI 是否理解编译期图与 Scope；模块化是否能讲依赖规则与治理工具；通信方案是否说得出失效边界。
  >
  > **加分**：Hilt Scope 层级、api/implementation、接口下沉、路由降级、Predictive Back 与 Navigation 适配。
  >
  > **减分**：只会说"分模块解耦"无依赖规则；推荐 EventBus 做主通信；把 DI 当"不用 new"这么浅。

---

* 大型 App 的进程/线程/存储分层架构（架构师视角）

  > **进程分层**：主 UI 进程（轻，只做展示与交互）、独立进程（:push/:webview/:download/:sandbox 隔离崩溃与内存）、可选多进程插件宿主。跨进程统一走 AIDL/ContentProvider + 序列化层，避免散落 Binder 调用。
  >
  > **线程分层**：主线程（UI/生命周期）、Default（CPU 核数协程池/线程池）、IO（网络磁盘弹性池）、业务专属串行池（DB 写、日志）。原则：**收敛线程数**，禁止每个模块私拉线程；统一入口（AppDispatchers）便于监控与替换。
  >
  > **存储分层**：内存缓存（LruCache/StateFlow）→ 磁盘（Room/DataStore/文件）→ 网络（分页/缓存策略 Cache-Control）；统一 Repository 屏蔽数据源，定义缓存一致性策略（先展示缓存再刷新）。
  >
  > 面试考察：能否把"一个请求从 UI 到磁盘/网络经过哪些层、线程在哪切、失败如何降级"讲成完整故事。

* Compose 与 View 体系共存与迁移策略

  > **共存**：ComposeView 嵌在 View 树（Fragment/Activity 布局中），AndroidView 反向嵌 View（WebView/MapView）。互操作是官方一等公民。
  >
  > **迁移策略**：
  > 1. 新页面 Compose，存量不动（渐进）。
  > 2. 复杂列表/表单先行（Compose 声明式优势区）；WebView/地图等重 View 组件用 AndroidView 包裹保留。
  > 3. 主题/设计系统先双轨（同一 Design Token 供 XML 与 Compose 使用）。
  > 4. 性能：Compose 有重组（recomposition）成本，避免在 recomposition 中做重逻辑（remember 稳定 lambda、stable 类型）。
  >
  > 面试追问："Compose 为什么不需要 findViewById？"——SlotTable/组合（composition）产出 UI 树，重组按状态变化局部更新；与 View 树+invalidate 机制本质不同。

* 稳定性（崩溃/ANR）治理体系怎么搭（不是接 SDK，而是流程）

  > **采集层**：Java UncaughtExceptionHandler + Native Breakpad/Crashpad + ANR（traces/ApplicationExitInfo）+ 卡顿（JankStats/插桩采样）。
  >
  > **聚合与归因**：堆栈符号化 → 按版本/设备/页面/用户分组聚类 → 新增崩溃 TOPN 排序 → owner 分派（模块负责人）。
  > **门禁与放量**：灰度分阶段放量，崩溃率/ANR 率对比基线，超阈值**自动回滚/暂停**；发布卡口（关键页面 monkey + 主流机型跑通）。
  >
  > **指标定义**：崩溃率（崩溃用户/DAU，分 Java/Native）、ANR 率、启动成功率；用分位数看而非均值。
  >
  > 面试区分度：能讲"从指标定义到回滚闭环"的，对齐稳定性平台工程；只会"接 Bugly"的停留在集成层。

* 启动优化的体系化方法论（链接 Practice 但架构层视角）

  > **度量**：点击→首帧可交互（分设备 P50/P90/P99）；Macrobenchmark 自动化；线上采集冷启动分布。
  >
  > **分段归因**：fork/attach（系统）→ Application.onCreate（三方 SDK 初始化重灾区）→ 首 Activity（布局/数据）→ 首帧（GC/measure）。
  >
  > **治理**：异步化+依赖分层（App Startup 管 DAG）、延迟非关键 init、Baseline Profile AOT 关键路径、启动窗口与真实布局一致（避免白闪）、页面级懒加载。
  >
  > **防劣化**：CI 跑 Macrobenchmark 基线对比，新增 SDK init 要评审"是否必须同步/主线程"。
  >
  > 架构意义：启动优化本质是"初始化依赖图治理"，与 DI、模块化收敛同源——模块化好的 App 启动 init 天然可按模块懒加载。

* 架构决策记录（ADR）与演进：架构不是一次定型

  > 好架构师面试会讲：当前架构的历史包袱（为什么还有 MVC 存量）、演进路径（先收敛通信再拆模块再上 DI）、**架构适应组织**（Conway 定律：模块边界对齐团队边界，减少跨团队协调）。
  >
  > 反例：不顾存量强推全量重构、不给灰度、不留双轨期。加分回答体现"风险可控的演进"而非"推倒重来"。

* 面试官视角

  > **考察点**：能否站在"万人协作、亿级用户"场景讲分层、迁移、治理与演进，而非单点技术。
  >
  > **加分**：线程收敛入口、Compose 重组陷阱、稳定性门禁闭环、Conway 定律、ADR 演进思维。
  >
  > **减分**：架构名词堆砌无落地细节；认为 Compose 可一步全量替换 View；启动优化只会"异步初始化"三字。
