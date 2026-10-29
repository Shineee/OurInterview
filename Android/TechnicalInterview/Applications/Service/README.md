# Service 面试题

* startService 与 bindService 的生命周期与混合使用的完整语义

  > **startService 路径**：context.startService → Service.onCreate（仅一次）→ onStartCommand（可多次，每次一个 intent）→ 除非 stopSelf/stopService，否则一直存活。onStartCommand 返回值决定被杀后行为：
  > - START_STICKY：被杀后重建，intent 为 null（适合持续任务）。
  > - START_NOT_STICKY：不重建（适合可丢弃任务）。
  > - START_REDELIVER_INTENT：重建并重传最后 intent（适合下载等必须完成的任务）。
  >
  > **bindService 路径**：bindService → onCreate → onBind 返回 IBinder → 客户端 onServiceConnected 拿 binder 通信 → 所有客户端 unbind 后 onUnbind → ondestroy。绑定是引用计数的，多个客户端可绑同一 service。
  >
  > **混合**：既 start 又 bind 时，必须既 stopService 又 unbind 才真正销毁——这是"Service 不会停"的高频排查点。
  >
  > 设计动机：start 生命周期跟"意图"（做一次事），bind 跟"连接"（IPC 通信），分离让同一 Service 可被多种方式复用。

* Service 跑在哪个线程？为什么？在 onStartCommand 里做耗时操作的后果

  > Service **运行在宿主进程的主线程**，没有独立线程。它只是组件生命周期容器，不代表后台执行环境。
  >
  > 后果：onStartCommand 阻塞主线程 → 输入事件 ANR（5s）/广播超时；网络/磁盘在主线程是性能红线。
  >
  > 正确模型：Service 内部自建 HandlerThread/IntentService 模式，或把任务委托给 WorkManager/协程 Dispatchers.IO。追问"那为什么还要 Service"——因为需要"组件级生命周期 + 前台优先级 + 客户端绑定接口"，纯后台计算用 WorkManager 即可。

* 前台服务（Foreground Service）的作用、实现与限制演进

  > **作用**：提升进程优先级（oom_adj 更高不易被 LMK 杀），适合用户可感知的持续任务（音乐、导航、下载、健康追踪）。
  >
  > **实现**：startForegroundService 启动 → 在 onCreate/onStartCommand 时限内（Android 12 前 5s，12+ 有额外限制）调用 startForeground(notificationId, notification)，通知常驻。Android 14+ 要求声明 foregroundServiceType（mediaPlayback/location/camera 等）并在 Manifest 权限对应。
  >
  > **限制演进**：
  > - Android 8+：后台应用不能 startService 拉起后台 Service，需 startForegroundService 或 JobScheduler。
  > - Android 12+：后台启动前台服务受限（需豁免场景如 SYSTEM_ALERT_WINDOW、精确闹钟等），防止滥用保活。
  > - Android 14+：类型必须匹配实际使用，类型不对会 SecurityException；部分类型需用户授权（位置等）。
  >
  > 面试追问："怎么合法保活？"——前台服务 + 用户可见通知是官方认可路径；其它黑科技（双进程守护、拉活）在国产 ROM 与 Google Play 政策下都不可靠/违规。

* bindService 的 AIDL 与 Messenger 两种 IPC 模式怎么选

  > **Messenger**：基于 Handler 的轻量串行 IPC，服务端单线程处理，无需写 AIDL。适合简单消息传递（不支持高并发、复杂类型）。
  >
  > **AIDL**：定义接口生成 Stub/Proxy，支持多线程（oneway、in/out/inout 方向、Parcelable 数据类型）。适合复杂跨进程 API（如系统服务、插件化）。
  >
  > 选型：消息级用 Messenger；需要"像调用本地方法一样"的跨进程接口用 AIDL。AIDL 注意：接口稳定版本管理（versioning）、binder 死亡（linkToDeath / DeathRecipient）、线程安全（Binder 线程池默认 16 线程）。

* Service 被系统杀死后的恢复策略

  > 1. START_STICKY + 前台通知：被杀后系统重建（但 intent null）。
  > 2. JobScheduler/WorkManager：持久化任务，进程死任务仍在调度队列，满足约束即重新拉起。
  > 3. AlarmManager 精确闹钟：setExactAndAllowWhileIdle 可在 Doze 中唤醒（注意配额）。
  > 4. JobInfo.Builder.setPersisted(true)：重启后仍调度（需 RECEIVE_BOOT_COMPLETED）。
  >
  > 现代推荐：短小可延迟任务 → WorkManager；持续用户可见 → 前台服务；系统级定时 → JobScheduler。不要依赖 Service.onStartCommand 的"重启"做关键业务（重建时机不可控）。

* 面试官视角

  > **考察点**：start/bind 双生命周期、线程模型、前台服务政策演进（8+/12+/14+），是否理解"Service 不是后台线程"。
  >
  > **加分**：onStartCommand 三种返回值语义、混合使用必须双停止、AIDL vs Messenger、WorkManager 替代路径。
  >
  > **减分**：以为 Service=后台线程；推荐双进程保活；不知道 Android 12 前台服务启动限制。

---

* IntentService 的工作机制、为何自动停止、Android 8+ 为何受限、现代替代

  > **机制**：内部 HandlerThread + ServiceHandler；onStartCommand 把 intent post 进工作线程队列 → onHandleIntent 串行执行 → 队列空后 stopSelf(startId) 自动销毁（只在处理最后一个消息时真正停，避免处理中被杀）。
  >
  > **8+ 受限**：后台应用不能 startService（IllegalStateException）；IntentService 作为后台启动受限；它运行完即停，不适合需要前台可见的长任务。
  >
  > **替代**：JobIntentService 曾用 JobScheduler 承接（已废弃路径），现官方推荐 **WorkManager**（OneTimeWorkRequest + 约束，持久化、可链式、可观察）。面试要能讲清"IntentService 的串行+自动停思想在 WorkManager 中延续"。

* AIDL 跨进程调用的完整链路（源码级追问）

  > 客户端调用接口方法 → 生成的 Proxy 类把方法参数写入 Parcel → transact(TRANSACTION_xxx) 经 **Binder 驱动**跨进程 → 服务端 Stub.onTransact 反序列化 → 调真实实现 → 返回值写回 reply Parcel → 驱动拷回客户端 → Proxy 反序列化返回。
  >
  > 关键点：
  > - **一次拷贝**：Binder 驱动用 mmap 让接收方直接映射发送方数据，省一次完整复制。
  > - **线程池**：服务端 Binder 线程池默认最多 16 线程（THREAD_MAX），onBind 返回的 binder 可能被多线程并发调用——实现要线程安全。
  > - **oneway**：异步，不阻塞客户端，无返回值；用于"发了就走"的回调。
  > - **DeathRecipient**：linkToDeath 监听对端进程死亡，处理服务断开重连。
  >
  > 面试区分度：能讲到 Binder 驱动与 mmap 一次拷贝的，直接对齐内核层深度。

* Service 的 started / bound / destroyed 状态机与内存回收优先级

  > Service 一旦 started，独立于客户端存在（除非 stop）；bound 时引用计数>0 存在；两者皆无则销毁。
  >
  > 进程优先级：有前台 Service > 有可见 Activity > 有 started Service（服务进程）> 缓存进程。所以"启动了 Service"比"纯后台缓存进程"更不易被杀，但仍是可回收的"服务进程"级。
  >
  > 排查"Service 不死"：`adb shell dumpsys activity services` 查看 started/bound 状态、client 计数、是否有 bind 未 unbind 的泄漏（Activity 泄漏常见连带 Service 泄漏）。

* 本地服务 vs 远程服务（进程内 vs 跨进程）设计考量

  > 同进程 bindService：仍走 Binder 但属"本地调用"优化路径（不跨进程，直接 Java 调用），适合模块解耦但不必真 IPC。
  >
  > android:process=":remote" 远程：真跨进程，需 AIDL/序列化；好处是隔离崩溃与内存，坏处是复杂度与延迟。
  >
  > 大型 App：核心服务（推送、账号、下载）常独立进程，主 UI 进程通过 AIDL/ContentProvider 与其通信；考虑 binder 缓冲 1MB 共享池与多客户端并发。

* Service 与 Activity 销毁联动：旋转屏幕时 Service 会重建吗

  > Service 归属进程而非 Activity；旋转只销毁重建 Activity，Service 不受影响（进程还在它就在）。
  >
  > 若 Activity destroy 时 unbindService，且无 started，Service 会销毁；若还 started 则继续跑。绑定型 Service 在配置变更中：新 Activity 实例重新 bind，旧实例 unbind——引用计数可能短暂为 0？不会，系统对配置变更的 bind 有处理，通常引用计数保持（旧 unbind 与新 bind 在同一消息序列完成）。面试可答"依赖 started/bound 计数，配置变更一般不导致 Service 重建"。

* 面试官视角

  > **考察点**：IntentService 源码思想与替代、AIDL 全链路、Service 状态机与优先级。
  >
  > **加分**：Binder 线程池并发、oneway、DeathRecipient、dumpsys 排查泄漏绑定。
  >
  > **减分**：把 IntentService 当现代方案；不知道 AIDL 底层走 Binder；认为旋转会杀 Service。
