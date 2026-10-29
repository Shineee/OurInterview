# Activity 面试题

* Activity 四种启动模式的本质区别，以及它们各自解决什么问题

  > **standard**：默认模式。每次 startActivity 都 new 一个新实例压入当前 Task 栈顶。设计动机是最简单直接——调用方不关心复用。问题：通知栏反复点开会无限叠实例。
  >
  > **singleTop**：栈顶复用。目标已在栈顶时不再创建，回调 onNewIntent。解决"通知/推送重复打开同一页面"的叠栈问题。注意：只查栈顶，若它在栈中间（上面还有别的 Activity），仍会新建实例压栈——这是高频陷阱。
  >
  > **singleTask**（官方语义 launchMode="standard" + taskAffinity 配合更准确的理解是"系统会为其寻找或创建匹配 taskAffinity 的 task"）：全局单实例 + 清栈。启动时若已存在，把该实例之上的所有 Activity 全部 destroy，将其带到栈顶并回调 onNewIntent。解决"App 内某核心页只能有一个，且返回时直接回桌面/上游"的产品需求。跨应用启动时会放入与 taskAffinity 匹配的 task。
  >
  > **singleInstance**：独占一个 Task，且该 Task 只有它一个 Activity。适合系统级入口如来电界面、来电展示页。其它应用再启动它时会复用同一实例并可能把请求转给 onNewIntent。
  >
  > 本质：四种模式回答的是同一个问题——"当请求到达时，系统如何决定【实例是否存在 + 放在哪个 Task + 栈内顺序】"。

* launchMode 与 taskAffinity 的关系，以及 singleTask 的真实行为（高频追问）

  > **taskAffinity** 决定 Activity"期望归属"的 Task 名，默认为应用包名。它只在特定 launchMode（singleTask、singleInstance 以及 allowTaskReparenting）与 Intent 带 FLAG_ACTIVITY_NEW_TASK 时才真正起作用，standard/singleTop 下系统一般忽略它。
  >
  > singleTask 的完整匹配逻辑：启动时先按 (taskAffinity, launchMode) 找已有实例；找到则清栈顶 + onNewIntent；找不到则新建实例，并放入与 affinity 匹配的 Task（若无则新建 Task）。
  >
  > **跨应用/多进程追问**：若 A 应用想以 singleTask 拉起 B 应用的 Activity，且不带 FLAG_ACTIVITY_NEW_TASK，会抛异常（startActivity 必须有 task 上下文）。带了 NEW_TASK 后，系统按 B 的 taskAffinity 为其新建/复用 Task。若 B 是多进程，实例在 B 的主进程内管理，Task 记录在系统 AMS 中。
  >
  > **Launcher 语义**：从桌面（Launcher）点击图标，系统以 Intent.CATEGORY_LAUNCHER + ACTION_MAIN 找到主 Activity，taskAffinity 决定它在哪个 Task 显示。若 taskAffinity 与包名不同且未在 Launcher 显示，会出现"从桌面点图标新建 Task、应用内已有 Task 另存一份"的双 Task 混乱——这正是很多"返回键回不到桌面/出现两次应用"问题的根因。
  >
  > 陷阱：很多人背"singleTask 就是单例"，但 singleTask 的"单实例"是指在**同一 taskAffinity 的 Task 内**单例，不同 taskAffinity 仍可能多份；真正的全局单例语义更接近 singleInstance（但它独占 Task）。

* onNewIntent 的调用时机与注意事项

  > 当系统选择复用已有实例（singleTop 栈顶命中、singleTask 清栈后、singleInstance、或显式 FLAG_ACTIVITY_SINGLE_TOP）时回调 onNewIntent 而非 onCreate。
  >
  > 注意点：
  > 1. onNewIntent 传入的 Intent 不会自动覆盖 getIntent()，需要手动 `setIntent(intent)`，否则后续 getIntent() 拿到的仍是旧 Intent——高频 bug。
  > 2. 此时 Activity 处于 resumed 之前的已创建状态，不要在这里重复初始化只应在 onCreate 做的重逻辑。
  > 3. 若 launchMode 为 singleTop/singleTask，任务栈记录的 Intent 更新依赖 setIntent，系统回收后重建时用的是栈内记录的 Intent。

* Intent Flag 与 launchMode 的优先级，常用 Flag 有哪些

  > **Flag 优先于 manifest 中的 launchMode**。系统先看 Intent 的 flags，再看 launchMode。例如 launchMode=singleTask 的 Activity 若被显式以 FLAG_ACTIVITY_CLEAR_TOP|FLAG_ACTIVITY_SINGLE_TOP 之外的方式启动，行为以 flag 组合为准。
  >
  > 常用 Flag：
  > - FLAG_ACTIVITY_NEW_TASK：在新 Task 中启动（或复用匹配 affinity 的 Task）。后台启动 Activity 时 Android 10+ 有严格限制（需已有 task 或系统豁免场景）。
  > - FLAG_ACTIVITY_CLEAR_TOP：清掉目标之上所有实例。与 SINGLE_TOP 同用则复用并 onNewIntent，不同用则 destroy 旧实例重建。
  > - FLAG_ACTIVITY_SINGLE_TOP：效果类似 singleTop launchMode，但只影响本次启动。
  > - FLAG_ACTIVITY_NO_HISTORY：不留在历史栈，退出即 destroy。
  > - FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS：不出现在最近任务列表。
  >
  > 面试追问："通知栏打开详情页用什么？"标准答案：singleTop + 在通知构建时用 FLAG_ACTIVITY_CLEAR_TOP|FLAG_ACTIVITY_SINGLE_TOP（或 TaskStackBuilder 处理返回栈），避免返回键穿越到通知场景外。

* 系统因内存不足杀死进程后，用户返回该 Activity 时发生了什么（异常终止恢复链）

  > 完整链路（面试官期待你讲清 AMS + 进程 + 状态恢复三层）：
  > 1. 后台进程被 LMK（Low Memory Killer，基于 oom_adj 分级）杀掉，AMS 移除进程记录但**保留 ActivityRecord（含 task 栈记录）**。
  > 2. 用户从最近任务或返回键回到该 Task，AMS 发现目标 ActivityRecord 的进程不在，走 **ActivityStarter → 冷启动该进程**（zygote fork）。
  > 3. 新进程起来后 AMS 绑定 Application，再按栈内 ActivityRecord **重建 Activity**：onCreate(savedInstanceState != null) → onStart → onRestoreInstanceState/onCreate 恢复 → onResume。
  > 4. 状态来源是之前 onSaveInstanceState 时 AMS 通过 `ActivityClientRecord.state` 保存的 Bundle（随 ActivityClientRecord 持久在 system_server 侧，进程死亡不丢）。
  >
  > 关键点：savedInstanceState 不是存在你自己的进程里，而是经由 AMS 保管——这就是"进程死了数据还在"的原因。面试若只答"会走 onCreate"而不提 AMS 保管状态，深度不够。

* Task、BackStack、ActivityRecord 的关系（源码层追问）

  > AMS（ActivityTaskManagerService）侧数据结构：**Task**（对应"任务/返回栈"，Android 10+ 重构为 TaskFragment/Task 层级）→ **ActivityRecord**（一个 Activity 实例在系统侧的令牌/记录，含 intent、state、app 进程引用）→ 进程侧的 ActivityClientRecord。
  >
  > Task 记录的是"逻辑栈"，与进程无关；进程死了 Task 还在，所以能恢复。返回键时 ActivityThread 收到 AMS 的 destroy 指令，弹栈到下一条 ActivityRecord。
  >
  > 最近任务（Recents）展示的就是这些 Task 的缩略图（任务快照 snapshot）。"从桌面进 App 再按 Home，Recents 显示的卡片"对应的就是这个 Task。

* 启动一个 Activity 的完整流程简述（framework 层）

  > 应用进程：startActivity → Instrumentation.execStartActivity → **ATMS（ActivityTaskManagerService，Binder 调用）**。
  >
  > system_server：ActivityStarter 计算启动模式/Flag/Task 归属（loadActivityLocked、computeLaunchingTaskFlags 等）→ 目标进程不存在则通过 AMS/ProcessList 让 Zygote fork → 目标进程 ActivityThread.main → attachApplication 与 AMS 建立 Binder 连接。
  >
  > 回到应用进程：AMS 通过 ApplicationThread（Binder）调度 → ActivityThread.handleLaunchActivity → performLaunchActivity → Instrumentation.newActivity 反射创建实例 → attach（绑定 Context/Window）→ onCreate。
  >
  > 视图上屏发生在 onCreate/onResume 设置 setContentView 之后的首次Traversal（Choreographer 下一帧 VSYNC 回调 measure/layout/draw），而非 onCreate 返回瞬间——这是"onCreate 里拿不到 View 宽度"的根因。

* 面试官视角

  > **考察点**：不是背四种模式名词，而是能否讲清"实例存在性、Task 归属、栈内顺序"三个决策维度，以及 AMS 侧 Task/ActivityRecord 如何支撑跨进程恢复。
  >
  > **加分回答**：提到 taskAffinity 与 Launcher 双 Task 问题、FLAG 优先于 launchMode、onNewIntent 需 setIntent、系统侧 ActivityRecord 保存 savedInstanceState、启动流程到 ATMS/ActivityStarter。
  >
  > **减分回答**：只会背"standard 新建、singleTop 栈顶复用……"一句口诀；把 singleTask 说成"全局单例"；不知道 singleTop 在非栈顶仍会新建；认为进程被杀后 Task 也一起没了。

---

* Activity 正常生命周期的五个状态与回调对应关系，为什么这样设计

  > 系统侧 ActivityRecord 有 ProcessState（进程是否存活）与 State（INITIALIZING/STARTED/RESUMED/PAUSING/STOPPED/DESTROYED）两层。应用侧回调与之对应：
  > - **CREATED（onCreate）**：实例创建、窗口（PhoneWindow）与视图树准备； savedInstanceState 非空表示是恢复重建。
  > - **STARTED（onStart）**：可见但未必可交互（可能被半透明层挡住）。
  > - **RESUMED（onResume）**：位于前台、接收输入、开始绘制第一帧。
  > - **PAUSED**：失焦（可能仍可见，如被对话框 Activity 遮挡）。
  > - **STOPPED**：完全不可见，窗口从窗口系统移除。
  > - **DESTROYED（onDestroy）**：实例销毁。
  >
  > **设计动机**：把"进程生死"与"界面可见性/焦点"拆成多级，是为了让系统能在不杀进程的前提下尽量回收资源（窗口、Surface、传感器），同时给应用精确的资源释放时机——单一 onCreate/onDestroy 两级无法表达"可见但不可交互"（如锁屏上方的通知面板）这种真实 UI 状态。

* 按 Home、锁屏、来电覆盖、分屏、转屏分别走什么生命周期（追问链）

  > - 按 Home：onPause → onStop。进程转入 cached（缓存进程），oom_adj 变小但仍存活。
  > - 锁屏：等价于失焦+不可见，onPause → onStop（大部分场景）。
  > - 全屏来电覆盖：被覆盖 Activity onPause → onStop（完全被不透明界面挡住）。
  > - 对话框样式的 Activity 覆盖：只 onPause，不 onStop（半透明、原 Activity 仍部分可见）。
  > - 分屏/多窗口：分屏中**两个 Activity 都处于 RESUMED**（Android N+ 支持多 RESUMED），各自 onPause/onResume 与窗口焦点联动，不能假设"只有前台一个 resumed"。
  > - 转屏（默认 configChanges 不含 orientation 时）：onPause → onStop → onSaveInstanceState → onDestroy → onCreate → onStart → onRestoreInstanceState → onResume，整树重建。
  >
  > 陷阱：很多旧答案说"分屏时后台那个是 paused"，N 之后多窗口下两个都可能 resumed——面试官用分屏考点就是在筛这个。

* onSaveInstanceState 的精确时机（Android 版本差异）与它到底存了什么

  > - Android 9（P）之前：onPause → **onSaveInstanceState** → onStop。
  > - Android 9 及之后：onPause → onStop → **onSaveInstanceState** → onDestroy。
  >
  > 调整动机（官方）：让状态保存发生在"确定不再可见之后"，避免与 UI 变更竞争。
  >
  > 存的内容：View 树的状态（EditText 文本、ListView 滚动位置、Checkable 状态）由 View 层自动写入（View.onSaveInstanceState，key 形如 android:viewHierarchyState），加上你手动 put 的 Bundle。整包经 Binder 交给 **system_server 中的 ActivityClientRecord.state 持有**——所以进程被杀后重建仍能拿到。
  >
  > 追问："Bundle 会不会太大导致 TransactionTooLargeException？"会。跨进程 Binder 事务缓冲约 1MB，把大 Bitmap 塞进 outState 是线上崩溃源。解法：只存轻量状态；大数据走 SavedStateHandle 自定义序列化或落盘。

* onSaveInstanceState 一定被调用吗？哪些场景不会？

  > 不一定。系统设计原则是"只为**可能需要恢复**的销毁保存状态"：
  > - 用户主动 finish（按返回键）：不调用（用户意图是离开，无需恢复）。
  > - 你在代码里显式调用 finish()：不调用。
  > - 进程在后台被 LMK 杀死：**会**先调用（如果还有机会在被杀前调度到），或更准确说：进入后台 onStop 前后已经保存过一次；若是在前台极端低内存被强杀（罕见），可能来不及保存。
  > - API 28+：即使 onStop 之后被杀，onSaveInstanceState 已在 onStop 后调用过。
  >
  > 面试陷阱：不要绝对化说"按 Home 一定走 / 返回键一定不走"，要落到"系统判断是否需要恢复"的设计意图上。

* onRestoreInstanceState 与在 onCreate 里读 savedInstanceState 的区别

  > 数据相同，时机不同：
  > - onCreate(bundle)：bundle 可能为 null（冷启动）；此时 View 树尚未 inflated，不能直接恢复 View 状态，只能恢复自定义字段。
  > - onRestoreInstanceState：在 onStart 之后调用，View 树已就绪，**super.onRestoreInstanceState 会自动把 viewHierarchyState 分发回各 View**。
  >
  > 实践：字段恢复放 onCreate 判断 null；依赖 View 的恢复（如 EditText）交给 onRestoreInstanceState 或让系统默认分发。若你在 onCreate 里手动恢复了 View 状态又没注意顺序，可能被后续分发覆盖——搞清谁先谁后是加分点。

* onConfigurationChanged 的正确用法与不推荐滥用的原因

  > Manifest 配置 `android:configChanges="orientation|screenSize|keyboardHidden|screenLayout|smallestScreenSize"` 后，转屏/分屏尺寸变化不再走销毁重建，而是回调 onConfigurationChanged(newConfig)，需自行 `recreate` 布局或切换资源。
  >
  > 为什么不推荐默认开启：
  > 1. 资源限定符系统（res/layout-land、values-night）是 Android 官方为"重建"设计的，开启 configChanges 后你等于自己实现一遍配置分发，容易漏改（深色模式、字体缩放、折叠屏展开等新维度越来越多）。
  > 2. 第三方库/子 View 若依赖"重建"来刷新配置，会失效。
  > 3. 合理场景：相机预览、视频播放、地图等**重建代价极高**的场景，或 WebView 需避免重新加载。
  >
  > 更现代的替代：用 Jetpack 的配置感知组件（AppCompatDelegate、WindowSizeClass、Compose 的本地配置），把适配做成"响应配置变化"而非"依赖重建"。

* 生命周期里做耗时操作会怎样？onCreate 与 ANR 的真实关系

  > onCreate/onResume 阻塞超过一定时间并不直接等于 ANR——**ANR 判定在输入事件与广播/Service 超时**：
  > - 输入事件：InputDispatcher 等待应用进程处理按键/触摸，主线程若卡在 onCreate 里导致 5s 内没处理 InputEvent，报 Input dispatching timed out。
  > - 广播：前台广播 10s / 后台 60s。
  > - Service：前台 20s / 后动 200s（前台服务 startForeground 有 5s 的 ANR，Android 12+ 前台服务启动限制另算）。
  >
  > 所以 onCreate 里 IO 3s 也许"不崩溃"，但会造成冷启动白屏时间长（Starting Window 消失前的观感卡顿）、首帧延迟、用户可感知的慢——这是启动优化里把"Application.onCreate + 首 Activity.onCreate"拆成异步/按需初始化的核心动机。

* 面试官视角

  > **考察点**：是否真正理解多级状态背后的设计动机（进程/可见性/焦点分离），能否处理分屏、版本时机差异、Binder 传状态等真实边界，而不是背"onCreate onStart onResume"。
  >
  > **加分回答**：讲清 savedInstanceState 存于 system_server、onSaveInstanceState 版本时机差异、分屏双 resumed、Binder 1MB 限制、ANR 判定与生命周期阻塞的关系。
  >
  > **减分回答**：只背口诀；认为 onRestoreInstanceState 是恢复的唯一入口；不知道返回键不保存状态；用"onCreate 不能耗时=会 ANR"这种不精确表述。

---

* Activity 间通信的完整方案谱系，以及各自的失效边界

  > 按"耦合度从紧到松"分层，面试要能说清每层的失效场景：
  >
  > 1. **Intent + Bundle/Parcelable**：点对点、一次性。失效边界：Binder ~1MB 缓冲（TransactionTooLargeException）；跨进程传大对象应改用文件路径/Uri/共享内存；被系统回收重建后 Intent 仍在栈记录里，但运行时对象无法还原。
  >
  > 2. **startActivityForResult → Activity Result API**：请求-响应式回传。旧 API 依赖 requestCode 匹配，fragment 嵌套时容易拿错；Jetpack `registerForActivityResult` 用类型化契约（ActivityResultContract）替代。注意注册时机必须在 STARTED 之前（onCreate/字段初始化），否则 IllegalStateException。
  >
  > 3. **共享 ViewModel（activityViewModels）**：同一 ViewModelStoreOwner 作用域内 Fragment 共享，或通过"activity 级 ViewModelStore"让两个 Activity 共享（不常见）。失效边界：进程被杀后 ViewModel 被清（除非 SavedStateHandle）；两个独立 Task/进程间无法自然共享。
  >
  > 4. **LiveData / SharedFlow / EventBus**：发布-订阅。失效边界：LiveData 依附 LifecycleOwner，宿主 destroy 后收不到（这是特性非缺陷）；EventBus 忘记 unregister 会泄漏且可能收到过期事件；全局 Flow 要用 Channel/SharedFlow(replay=0) 避免粘性事件歧义。
  >
  > 5. **单例/静态**：进程级共享。失效边界：多进程下每个进程一份实例（"单例不单"）；生命周期无人管理。
  >
  > 选型原则：能用 Intent 一次性带完的不用共享；同作用域优先共享 ViewModel；跨页面/跨模块用事件总线或路由框架，并明确注销/生命周期。

* Parcelable 与 Serializable 的底层差异，为什么 Parcelable 快

  > Serializable 走 Java 对象序列化：依赖反射扫描字段、写 ObjectStream，产生大量临时对象，API 简单但 CPU/内存开销大，且 serialVersionUID 不匹配会 InvalidClassException。
  >
  > Parcelable 直接把字段按顺序写入 Parcel（一块可跨进程的字节缓冲），读时按同样顺序取出——**省掉反射与中间对象**，所以官方称快约 10 倍。代价是手写模板长（Kotlin @Parcelize 可生成）。
  >
  > 底层：Parcel 数据最终经 Binder 驱动拷贝到目标进程（一次拷贝，mmap 接收缓冲），所以"快"也意味着你可控地决定了跨进程传输的字节布局。Serializable 的兼容性（ serialVersionUID）在"需要长期稳定跨版本"场景仍有价值；Android 组件间通信默认 Parcelable。

* 为什么 Intent 传大数据会崩？Binder 事务缓冲的精确限制

  > 每个进程的 Binder 缓冲默认约 1MB（`/dev/binder` 的 mmap 大小，1MB-8KB 量级），是该进程所有 Binder 事务共享的。单个事务（如 startActivity 携带的 Bundle 序列化后）过大，或并发多个大事务挤占，抛 **TransactionTooLargeException**。
  >
  > 注意：这个 1MB 是**进程级共享池**，不是"单个 Bundle 上限 1MB"——你的 App 越复杂、同时进行的 Binder 调用越多，单次可用余量越小。
  >
  > 真实排查：`TransactionTooLargeException` 栈常指向 Parcel.writeBundle，解决思路永远是"别把大 payload 塞进 Intent/Bundle"：传 Uri（ContentResolver 按需读）、文件路径、或用共享内存（ashmem/MemoryFile，如系统剪贴板大文本）。

* 静态变量/单例在多进程下的陷阱

  > 若 App 声明 `android:process=":remote"`，每个进程有独立的虚拟机与堆，**static 字段各进程独立初始化**——A 进程 put 进静态 Map 的数据，B 进程看不到。
  >
  > "单例"同样每进程一份。若你的组件设计依赖"全局单例同步状态"，多进程即失效。
  >
  > 正确跨进程方案：ContentProvider（封装跨进程查询/更新，走 Binder）、AIDL Service、或落到数据库/文件 + 各进程监听变更（ContentObserver / FileObserver）。ContentProvider 是最轻的"系统帮你管理 Binder"的方式——很多 SDK（如 WorkManager 初始化、微信 SDK）用它做跨进程入口就是这个原因。

* 两个 Activity 之间"同步"拿到结果的正确姿势（含协程封装）

  > 直接"启动 B 并阻塞等 B 结果"在主线程是死锁（B 的 onCreate 也需要主线程）。正确异步模型：
  > ```
  > // Activity A
  > val launcher = registerForActivityResult(StartActivityForResult()) { result ->
  >     if (result.resultCode == RESULT_OK) {
  >         val data = result.data
  >     }
  > }
  > launcher.launch(intent)
  > ```
  > 协程可把回调再包一层 suspendCancellableCoroutine，让调用方像同步一样写：
  > ```
  > suspend fun awaitResult(intent: Intent): Intent? =
  >     suspendCancellableCoroutine { cont ->
  >         val l = registerForActivityResult(...) { r -> cont.resume(r.data) }
  >         l.launch(intent)
  >     }
  > ```
  > 注意 launcher 的生命周期与取消语义；不要在协程里真的阻塞线程。

* 面试官视角

  > **考察点**：是否踩过 Binder 缓冲、多进程静态变量、EventBus 泄漏这些真实坑，还是只会列 API 名字。
  >
  > **加分回答**：给出"1MB 是进程级共享池"的精确理解；提到 ContentProvider 作为跨进程通信的轻方案；Activity Result API 的注册时机约束。
  >
  > **减分回答**：只背"Intent 可以传对象"；不知道 Parcelable 与 Binder 拷贝的关系；推荐用静态变量在 Activity 间共享状态。

---

* 横竖屏切换时系统到底做了什么？（默认与 configChanges 两条路径）

  > **默认路径（推荐）**：
  > 1. 旋转触发 Configuration changed（orientation、screenLayout、smallestScreenSize 等）。
  > 2. AMS 判断 Activity 的 manifest 未声明能处理的 configChanges → 走"销毁重建"：onSaveInstanceState → onDestroy → 新实例 onCreate(savedInstanceState 非空)。
  > 3. 资源系统按新 Configuration 重新选择资源（layout-land vs layout、layout-sw600dp 等），setContentView 加载的是新限定符目录下的布局。
  >
  > **configChanges 路径**：Manifest 声明 `android:configChanges="orientation|screenSize"` 后，不销毁，回调 onConfigurationChanged(newConfig)，应用自己切布局/更新尺寸。
  >
  > 设计动机：重建让"资源限定符 + 状态保存"自动工作，代价是整棵 View 树与 Controller 重建；configChanges 省重建但把适配责任全压给应用——折叠屏/窗口尺寸/字体缩放等维度爆炸后，官方更倾向"响应式适配"（WindowSizeClass）而非无限往 configChanges 里堆 flags。

* onConfigurationChanged 里应该做什么？常见漏改项

  > 至少做：
  > 1. 根据 newConfig.orientation 切换布局或更新布局参数。
  > 2. 通知依赖尺寸的自定义 View invalidate/remeasure。
  > 3. 若用了 WebView/相机/地图，切换预览方向或刷新。
  >
  > 常见漏改（面试官爱问"你开了 configChanges 之后线上出了什么坑"）：
  > - 深色模式（uiMode）没配，夜间切换仍重建或不刷新。
  > - 字体缩放（fontScale）与屏宽（screenSizeDp/smallestScreenSizeDp）在分屏拖动时持续变化，只配 orientation 不够。
  > - 第三方 Fragment/库内部缓存了旧尺寸未通知。
  >
  > 现代做法：能不自己处理配置就交给系统重建；必须长生命周期的（播放器）用 ViewModel 持数据 + View 层响应重建。

* onSaveInstanceState 保存自定义对象的正确姿势

  > 对象需 Parcelable/KSerializable，且类版本变化要处理（Parcelable 无内建版本兼容，字段增删可能反序列化错位；可用 Serializable + serialVersionUID 换兼容性）。
  >
  > 推荐模式： onSaveInstanceState 只存"恢复所需的最小 id/key"，重数据在 onCreate 用 id 从磁盘/DB 重载——这比把整个对象塞 Bundle 更抗系统回收与进程死亡。
  >
  > ViewModel + SavedStateHandle 组合：SavedStateHandle 内部就是 Bundle，但作用域跟 ViewModel 一致，且可在进程死后恢复，是 Jetpack 官方推荐替代手写 onSaveInstanceState 的方式。

* 进程被杀与转屏重建，两种 savedInstanceState 恢复有什么差异？

  > 相同点：都通过 Bundle + viewHierarchyState 恢复 UI 状态，onCreate 参数都非空。
  >
  > 差异：
  > - 转屏重建：同进程内，onSaveInstanceState 与 onCreate 几乎背靠背，恢复及时；ViewModel **不会**被清（ViewModelStore 在配置变更时保留，onRetainNonConfigurationInstance）。
  > - 进程被杀：整个进程的静态、ViewModel、堆对象全没；恢复时 Application.onCreate 也会重跑；ViewModel 需 SavedStateHandle 才能恢复。
  >
  > 面试官用这个对比考察你是否理解"配置变更保留"与"进程死亡清空"是两套不同机制（ActivityClientRecord.state vs ViewModelStore vs 进程堆）。

* 旋转导致的"白屏/闪一下"从哪来？排查路径

  > 根因：销毁重建期间旧窗口移除、新窗口 measure/layout/draw 完成前的间隙，系统用 Starting Window/Background 填充；若 onCreate 耗时长，间隙被拉长为可感知白屏。
  >
  > 排查：
  > 1. `adb shell dumpsys activity activities` 看是否走 destroy+recreate。
  > 2. Profiler 看主线程 onCreate 耗时与 GC。
  > 3. 检查是否在 onCreate 同步 inflate 大布局/读盘。
  >
  > 缓解：configChanges（有限场景）、预 inflate、把重初始化移出首帧路径、用 Compose/更轻的重建单元。

* 分屏/自由窗口下 onResume 不止一个，对你意味着什么

  > Android N+ 多窗口：可见的两个 Activity 可同时处于 RESUMED；窗口焦点变化会各自回调 onPause/onResume（获得/失去键盘输入焦点），但**可见性（start/stop）与焦点（pause/resume）解耦**了。
  >
  > 影响：依赖"onResume 即唯一前台"的逻辑（如相机抢占、播放器自动播放、传感器注册）在分屏下会两个页面都触发。正确做法：用 Lifecycle 的 RESUMED + 焦点感知（View.isFocused / Window 的 focus 回调）或仅在真正需要时抢占独占资源。

* 面试官视角

  > **考察点**：是否理解转屏"销毁重建"是资源系统驱动的特性而非缺陷；能否对比配置变更保留（ViewModel）与进程死亡（全清）两条恢复路径。
  >
  > **加分回答**：讲清资源限定符在重建中的作用、SavedStateHandle 相比手写 Bundle 的优势、分屏双 RESUMED 对单例资源逻辑的冲击。
  >
  > **减分回答**：认为转屏白屏是"系统 bug"；把 ViewModel 说成进程被杀也能活；推荐所有页面都开 configChanges。

---

* Task 与返回栈（Back Stack）的真实模型，Android 10+ 的重构

  > 传统模型：Task = 一叠 ActivityRecord，按启动顺序入栈，返回键弹栈。Task 有 taskId、taskAffinity、rootActivity。
  >
  > Android 10 引入 **TaskFragment / ActivityEmbedding** 重构：把"任务"拆成可嵌套的 TaskFragment，支持双栏并排（Jetpack WindowManager / Activity Embedding），一个 Task 内可同时容纳两个 Activity 呈现左右分栏。返回栈语义扩展为 TaskFragment 内部栈 + Task 级栈。
  >
  > 面试意义：说明"返回栈"不再是一维列表，折叠屏/大屏适配（官方 WindowSizeClass、embedding 库）正基于此。提到这一层说明你关注演进而非背旧模型。

* 从桌面图标到 Activity 上屏：完整启动流程（分进程讲）

  > **1. Launcher 进程**：点击图标 → Launcher startActivity（Intent ACTION_MAIN + CATEGORY_LAUNCHER）→ Binder 到 system_server。
  >
  > **2. system_server（ATMS/ActivityStarter）**：解析 Intent → 查找目标 ActivityRecord（含权限、IntentFilter 匹配）→ 计算 launchMode/Flags/Task 归属（ActivityStarter.execute → startSpecificActivity）→ 目标进程不存在则 Zygote fork（通过 AMS 的 ProcessList + socket 通知 zygote）。
  >
  > **3. 新进程（应用）**：Zygote fork → ActivityThread.main → prepareMainLooper → attachApplication（Binder 回 AMS，带上 ProcessName/ApplicationThread）→ AMS 绑定 ApplicationInfo → handleBindApplication（创建 Application、ContentProvider init、调 Application.onCreate）→ 再按待启动的 ActivityClientRecord 执行 handleLaunchActivity。
  >
  > **4. Activity 创建**：performLaunchActivity → Instrumentation.newActivity（反射）→ Activity.attach（ContextImpl、PhoneWindow、WindowManager）→ onCreate → setContentView 构建 DecorView → onResume → ViewRootImpl setView → 首帧 measure/layout/draw（Choreographer VSYNC）。
  >
  > 优化视角：冷启动耗时 = 进程 fork + Application.onCreate + 首 Activity.onCreate + 首帧。"启动优化"就是压缩这条链上每一段的同步工作（异步 init、延迟加载、Baseline Profile、窗口预览）。

* Zygote 为什么用 fork 而不是普通进程创建？对 App 启动意味着什么

  > Zygote 预加载了 framework 类与资源（preloadClasses、preloadResources），fork 时**写时复制（COW）** 子进程立即共享这些页——这是 Android 秒级拉起几十个 App 的基础。
  >
  > fork 在 Zygote 主线程（单线程）进行，避免多线程 fork 的锁状态复杂性；子进程起来即具备已初始化的 ART 虚拟机与预加载类，应用进程只需加载自己的 dex。
  >
  > 面试可延伸：fork + COW 解释了"为什么 App 内存中有一大片与其他 App 相同的共享只读页"；也解释了 WebView 多进程/沙箱设计的起点。

* 进程优先级（oom_adj 分级）与回收顺序

  > AMS 给每个进程打 oom_adj 分数，LMK（内核 lowmemorykiller）按分数从高到低杀：
  > - **前台进程**（用户正在交互的 Activity/前台 Service）：最不易杀。
  > - **可见进程**（可见但非前台，如分屏另一侧）。
  > - **服务进程**（startService 后台服务）。
  > - **缓存进程**（后台 Activity，仅保留供返回快速恢复）：优先回收对象。
  > - 空进程：无活跃组件，最优先回收。
  >
  > Android 10+ 引入 **App Standby Bucket / 进程管理器（_cached/no-kill）** 等演进，部分 OEM 更激进。面试要能说"LMK 依据 oom_adj + 内存压力（PSI）"，而不是笼统"后台就杀"。

* 双击返回退出、返回键不出栈等"返回栈异常"的排查

  > 常见根因：
  > 1. singleTask/singleInstance + taskAffinity 配置不当，启动时新建了 Task，返回键留在"错误"的栈。
  > 2. 通知栏/外部拉起用了 FLAG_ACTIVITY_NEW_TASK 但没配好 back stack（应用内应使用 TaskStackBuilder 或 setTaskDescription）。
  > 3. onNewIntent 未 setIntent，getIntent 数据陈旧。
  > 4. launchMode=standard + 外部反复拉起，栈内堆叠多个实例，返回要按很多次。
  >
  > 排查工具：`adb shell dumpsys activity activities`（看 taskId、栈内 ActivityRecord 顺序、launchMode）、`adb shell dumpsys window`、Layout Inspector 看 Fragment 栈。

* 面试官视角

  > **考察点**：能否把"一个点击"讲到 ATMS/ActivityStarter/Zygote/首帧的全链路，并链接到启动优化与内存回收的产品指标。
  >
  > **加分回答**：Zygote COW 预加载、oom_adj 分级与 LMK、Android 10 TaskFragment 演进、dumpsys 排查返回栈。
  >
  > **减分回答**：只会背"点击图标走 startActivity"；不知道 Application.onCreate 在链路中的位置；说"返回栈就是 Activity 列表"而不提系统侧 ActivityRecord。
