# Android 异步面试题

* Handler 机制的完整链路：从 sendMessage 到 handleMessage，源码层怎么走的

  > **四个角色与职责**：
  > - Thread（线程）：消息循环的宿主。一个线程 Looper.prepare 后才有 MessageQueue。
  > - Looper：`ThreadLocal` 绑定到线程（`Looper.sThreadLocal`），持有唯一 MessageQueue，loop() 无限循环取消息。
  > - MessageQueue：单链表按 when 排序的消息队列。**并不是队列（Queue），是按时间排序的链表**。native 层用 epoll 监听管道（mMessages == null 时 nativePollOnce 阻塞，fd 是 wake pipe）。
  > - Handler：发送与处理的双重身份。sendMessage 时把 Message.target 指向自己；Looper 取出后 `msg.target.dispatchMessage(msg)` 回到对应 Handler。
  >
  > **发送链**：Handler.sendMessage → sendMessageDelayed → sendMessageAtTime → enqueueMessage：按 when 插入链表；若插在头部则 nativeWake 唤醒 epoll。
  >
  > **处理链**：Looper.loop()（主线程在 ActivityThread.main 中调 prepareMainLooper + loop）→ queue.next()（可能阻塞在 nativePollOnce）→ msg.target.dispatchMessage → 回调。
  >
  > **设计动机**：单线程 UI 模型下用"消息 + 队列"把跨线程调用变成同线程任务排队，省去锁 UI 控件的开销；epoll 阻塞让空闲时主线程不耗 CPU。

* MessageQueue.next() 与同步屏障（sync barrier）、async message 的关系（源码级追问）

  > next() 关键逻辑（简化）：
  > ```
  > if (现在时间 >= msg.when) 取出返回
  > else 计算 epoll 等待时长，nativePollOnce(timeout)
  > 若有屏障（mNextBarrierToken != 0 / postSyncBarrier 曾插入一条 target==null 的特殊消息）：
  >     只取 **async message**（msg.isAsynchronous()==true），忽略同步消息
  > ```
  > **同步屏障**是一条 `target == null` 的 Message，通过 `MessageQueue.postSyncBarrier` 插入（隐藏 API），作用是"在其存在期间，同步消息被拦住，优先处理异步消息"。
  >
  > **谁在用**：ViewRootImpl 的 scheduleTraversers 会 postSyncBarrier，然后通过 Choreographer 在 **VSYNC 到来时** post 一条 async 的 doFrame 消息，确保绘制帧消息优先于普通 Handler 消息被处理，降低掉帧概率。执行完 traversal 后 removeSyncBarrier。
  >
  > **面试陷阱**：屏障是隐藏 API，应用层不能合法调用（反射可调但灰色）；屏障必须配对 remove，否则同步消息永远得不到处理→"消息假死"。

* 为什么 Looper.loop() 死循环不会导致 ANR 或 CPU 100%

  > 1. ANR 的判定主体是 **system_server 侧的超时监控**（InputDispatcher / BroadcastQueue / ActiveServices），不是"主线程在 loop 里"。loop 空转不处理输入才会触发输入 ANR。
  > 2. CPU：queue.next() 在无消息时走 nativePollOnce → **epoll_wait 阻塞**，线程让出 CPU，占用接近 0。有消息才被 nativeWake（往管道写一字节）唤醒。
  > 3. 设计哲学：主线程本质是"事件循环"进程模型（类似 iOS RunLoop / 浏览器主线程），空闲即睡眠是正确行为；若主线程 loop 退出，应用 UI 直接死亡。
  >
  > 追问："Looper.loop 里 if (msg == null) return 是怎么回事？"——quit 时 next() 返回 null，loop 退出；正常运行永不为 null。quit/quitSafely 区别：后者会处理完已到期消息再退，前者立即退出。

* IdleHandler 的执行时机与典型用途

  > 在 MessageQueue.next() 中，当**当前没有到期消息需要立即处理**（即即将进入 epoll 阻塞）时，会先遍历 mIdleHandlers 执行 `queueIdle()`：
  > - 返回 false：执行一次后移除。
  > - 返回 true：保留，每次空闲都执行。
  >
  > 时机本质："主线程当前空闲，下一帧/下一条消息还没到"。常用于：延迟初始化非关键 SDK、GC 前的轻量任务（WeakReference + IdleHandler 做无用对象回收）、启动优化里把非 UI 初始化挪出关键路径（注意 idle 时机可能在首帧之后，不适合影响首屏的任务）。
  >
  > 陷阱：IdleHandler 不是"一定在首帧前"，若消息队列一直忙可能迟迟不执行；也不要在这里做长任务（会推迟后续消息处理造成掉帧）。

* Handler 引起内存泄漏的原理与正确写法

  > 泄漏链：非静态内部类 Handler 持有外部 Activity 引用 → postDelayed 的 Message 在 MessageQueue 中持有 msg.target（Handler）→ 消息未处理完前 GC 无法回收 Activity。
  >
  > 即使 Activity destroy，消息还在队列里"钉住"整个对象链。
  >
  > 正确做法：
  > 1. 静态内部类 + WeakReference<Activity>。
  > 2. onDestroy 里 `handler.removeCallbacksAndMessages(null)`（清空该 handler 所有消息；注意若用全局Looper，null token 会清掉所有同 target 消息）。
  > 3. Kotlin 协程/RxJava 场景改用 viewModelScope/lifecycleScope，生命周期自动取消。
  > 4. 架构层：状态下发用 LiveData/SharedFlow，避免长生命周期消息持有短生命周期界面引用。

* Handler、Looper、MessageQueue 与线程的关系；子线程能直接 new Handler 吗

  > 一个线程最多一个 Looper（prepare 两次抛 RuntimeException "Only one Looper may be created per thread"），一个 Looper 一个 MessageQueue，多个 Handler 可共享同一 Looper（各自靠 callback 区分消息）。
  >
  > 子线程 new Handler() 前必须 `Looper.prepare()`，否则抛 "Can't create handler inside thread that has not called Looper.prepare()"——因为 Handler 构造函数要 `mLooper = Looper.myLooper()`，而 myLooper 读 ThreadLocal，此时为 null。
  >
  > 主线程特殊：ActivityThread.main 调 `Looper.prepareMainLooper()`，全局唯一；`Looper.getMainLooper()` 可在任意线程拿到主线 Handler 用来切回主线程。
  >
  > HandlerThread = 封装了 Looper.prepare/loop 的线程 + 暴露 getLooper 的 Handler 宿主，省去手动 prepare。

* 面试官视角

  > **考察点**：是否读过 MessageQueue.next/Looper.loop 源码，能否讲清 epoll 阻塞、同步屏障与异步消息、IdleHandler 时机，以及泄漏的对象引用链。
  >
  > **加分回答**：主动提 ViewRootImpl + Choreographer 用屏障保证 VSYNC 帧优先；讲清 nativePollOnce/nativeWake 的 epoll 与 wake pipe；给出 removeCallbacksAndMessages 的清理时机。
  >
  > **减分回答**：把 MessageQueue 说成先进先出队列；认为 loop 死循环会占满 CPU；只答"静态内部类防泄漏"却讲不清引用链；不知道 async message 与屏障的关系。

---

* AsyncTask 为什么被废弃？它的线程模型与陷阱（面试常问历史题）

  > **模型**：SerialExecutor 串行执行（内部是 THREAD_POOL_EXECUTOR 的变体，早期版本并行有竞态），主线程 Handler post 结果。onPreExecute（主线程）→ doInBackground（后台）→ onPostExecute（主线程）。
  >
  > **废弃原因（官方）**：
  > 1. 生命周期无感知——旋转/finish 后仍回调，内存泄漏与窗口泄漏（Activity 已 destroy 还更新 UI）。
  > 2. 串行默认导致"一个慢任务挡住所有后续任务"（历史版本）；并行执行器又易被误用造成线程泛滥。
  > 3. 与协程/RxJava 相比缺少取消、结构化并发、可组合性。
  >
  > **残留陷阱**（老代码维护面试会追问）：
  > - execute() 必须主线程调用（内部 checkMainThread）。
  > - 结果回调持有 Activity 弱引用但**仍可能在 destroy 后送达**（WeakReference 只防 Activity 被泄漏到 VM 外，不防"逻辑上已不需要"）。
  > - cancel 后 doInbackground 仍可能跑完，isCancelled 需手动检查。
  >
  > **迁移答案**：lifecycleScope + Dispatchers.IO + withContext 回主线程，或 ViewModel + Flow。

* HandlerThread 的本质与典型使用场景

  > HandlerThread = 一个自带 Looper.prepare/loop 的 Thread + getLooper()。创建后需等 run() 里 prepareMainLooper 完成（用 CountDownLatch 或 wait looper 就绪）才能安全 getLooper。
  >
  > 典型用途：需要**串行化**执行的后台任务（IntentService 的前身思路、相机/Sensor 数据处理、SQLite 专用线程避免多线程抢连接）。比裸 Thread + 自建队列更省事，比线程池更适合"严格 FIFO 且单线程"场景。
  >
  > 追问："IntentService 与 HandlerThread 关系？"IntentService 内部就是 HandlerThread + Handler 处理 onHandleIntent，串行处理多个 startService 请求后自动 stopSelf。

* IntentService 的工作机制、为何自动停止、为何在 Android 8+ 受限

  > **机制**：onCreate 时创建 HandlerThread + ServiceHandler；onStartCommand 把 intent 消息 post 进队列 → onHandleIntent 在**工作线程**串行执行 → 队列空后 stopSelf(id) 自动销毁。
  >
  > **自动停止**：stopSelf(startId) 只在"当前处理的是最后一个消息"时才真正销毁，避免处理中被杀——这是它比普通 Service 写起来安全的核心。
  >
  > **Android 8+ 限制**：后台应用不能 startService（IllegalStateException），IntentService 作为后台 Service 启动受限；且它运行完即停，不适合"需要前台可见"的长任务。**JobIntentService**（已废弃路径）曾用 JobScheduler 承接，现官方推荐 WorkManager。
  >
  > 面试要点：IntentService 的"串行 + 自动停"思想在 WorkManager 里延续（OneTimeWorkRequest + 约束）。

* JobScheduler / WorkManager 与 Service 的选型

  > - **前台 Service**：用户可感知的持续任务（音乐播放、导航、通话）——必须发通知，系统杀得少。
  > - **普通 Service**：仅适合"被绑定"或短后台任务，Android 8+ 后台启动受限。
  > - **JobScheduler/JobService**：系统级调度，支持约束（网络/充电/空闲），延迟/周期，Doze 也排队执行；Android 5.0+。
  > - **WorkManager**：Jetpack 统一封装，低版本回退 JobScheduler/AlarmManager，支持链式/唯一任务/观察状态（LiveData/Flow），**默认持久化**（进程死任务还在）。
  >
  > 选型口诀：用户看得见→前台 Service；可延迟可约束的后台→WorkManager；精确到毫秒的闹钟→AlarmManager（setExactAndAllowWhileIdle，注意 Doze）。

* Service 跑在主线程吗？为什么？在 Service 里做网络请求会怎样

  > Service **没有独立线程**，onCreate/onStartCommand/onBind 都在**主线程**（与宿主进程主线程同一 Looper）执行。系统不给 Service 自动开线程，因为 Service 只是"组件生命周期容器"，不代表"后台线程"。
  >
  > 在 onStartCommand 里做网络：轻则 ANR（主线程 Binder/IO 阻塞输入与广播超时），重则被系统限制。正确：内部 HandlerThread/IntentService 模式、或 onHandleIntent 类工作线程，或交给 WorkManager。
  >
  > 追问陷阱："Service 不是后台吗？"——很多初级答案误以为 Service=后台线程，这正是此题筛选点。

* 面试官视角

  > **考察点**：对"官方废弃组件"是否理解其设计缺陷与现代替代，能否把 Service 的线程模型讲对。
  >
  > **加分**：IntentService 的 stopSelf(startId) 语义、WorkManager 持久化与回退链、JobScheduler 约束模型。
  >
  > **减分**：以为 Service 自动开线程；推荐用 IntentService 做 Android 12+ 的后台任务；只会背 AsyncTask 三个回调名字。

---

* ThreadPoolExecutor 核心参数与状态流转（源码级）

  > 构造参数：
  > - **corePoolSize**：核心线程数（即使空闲也保留，除非 allowCoreThreadTimeOut）。
  > - **maximumPoolSize**：最大线程数。
  > - **keepAliveTime**：非核心线程空闲存活时间。
  > - **workQueue**：BlockingQueue，核心满后任务入队。
  > - **threadFactory**：命名线程（排查必备）。
  > - **handler**：拒绝策略。
  >
  > **执行顺序（背源码逻辑而非名词）**：
  > 1. 当前线程数 < core → 创建核心线程执行。
  > 2. 核心满 → 任务入队（成功则不创建线程）。
  > 3. 队列满且线程数 < max → 创建非核心线程执行。
  > 4. 队列满且线程数 = max → 触发拒绝策略。
  >
  > **高频陷阱**：LinkedBlockingQueue 无界时，线程数永远停在 core，maximumPoolSize 失效 → 任务堆积 OOM。这也是"newFixedThreadPool 可能 OOM"的根源（内部就是 core=max 的无界队列）。
  >
  > 状态：ctl 高 3 位存 running/shutdown/stop 等，低 29 位存线程数；shutdown 只拒绝新任务处理存量，shutdownNow 中断并清空队列。

* 四种拒绝策略与自定义策略

  > - AbortPolicy（默认）：抛 RejectedExecutionException——快速失败，让调用方感知。
  > - CallerRunsPolicy：提交线程自己执行——**限流**效果（提交线程被占用），注意若提交线程是主线程会造成卡顿。
  > - DiscardPolicy：静默丢弃——日志要有，否则丢任务无感知。
  > - DiscardOldestPolicy：丢队列头最老任务，重试提交当前。
  >
  > 自定义：结合业务做"落盘待重试"或"降级到更大的缓冲"。面试可联系线上"任务被拒导致消息丢失"排查：先看是不是无界队列思维在有界池里用了默认拒绝。

* execute 与 submit 的区别，Future 与异常传播

  > execute(Runnable)：无返回值，异常在 worker 线程 uncaught → 线程死掉（池会补新线程），**调用方无感知**。
  >
  > submit(Callable/Runnable)：返回 Future；异常被包装进 FutureTask，get() 时抛 ExecutionException。**不 get 就永远看不到异常**——静默失败经典坑。
  >
  > CompletableFuture / Kotlin 协程 异常传播更结构化（协程里默认取消整个 scope，除非 SupervisorJob）。

* 常见线程池模板与参数怎么定（不要背"CPU 核数+1"）

  > 经验公式只是起点，必须结合任务类型压测：
  > - **CPU 密集**（计算、序列化）：core ≈ 核数（或核数+1 留一点调度余量），队列有界，避免过多线程切换。
  > - **IO 密集**（网络、磁盘）：等待时间长，可远大于核数（如 2×核数 起步，按 RT/QPS 压测），队列有界 + 合理拒绝。
  >
  > Android 特殊性：主线程之外常用"共享业务线程池"（如 Dispatchers.Default 本质是 CPU 核数线程池）；图片库/网络库各持有自己的池，面试可对比 OkHttp Dispatcher 与 Glide 的 Executors。
  >
  > 排查：jstack/Profiler 看池线程状态（WAITING 在队列 take 是正常的；RUNNABLE 长期跑说明池太小或任务太重）。

* 线程池在 Android 中的实践坑

  > 1. 线程名不规范 → 排查困难；threadFactory 统一命名前缀+序号。
  > 2. 在池里操作 View/Activity → 必须切主线程（Handler/withContext(Dispatchers.Main)）。
  > 3. 池任务持 Activity 引用 → 泄漏；任务结束前 Activity 销毁要可取消。
  > 4. 多池泛滥（每个 SDK 自带池）→ 线程总数失控，统一收敛共享池。
  > 5. shutdown 时机：单例池随进程活；页面级临时池必须 shutdownNow，防泄漏。
  > 6. 并发修改共享集合 → 用并发容器或不可变快照，不要在任务里裸 HashMap。

* 面试官视角

  > **考察点**：是否理解"核心-队列-最大-拒绝"的执行顺序与无界队列陷阱，能否给可压测的参数策略。
  >
  > **加分**：ctl 位运算状态、CallerRuns 限流副作用、execute 静默异常 vs submit 吞异常、Android 多池治理。
  >
  > **减分**：把 core/max 顺序记反；推荐无界队列；只会背"IO 密集用 2N+1"而不讲验证方法。

---

* suspend 函数的本质：编译后变成了什么（必考）

  > **不是线程切换魔法，而是 CPS 变换 + 状态机**。编译后：
  > - 函数签名加 `Continuation` 参数（`suspend fun foo(): Int` → `foo(continuation): Object`），返回 Object 用于协程挂起/恢复协议（COROUTINE_SUSPENDED 标记）。
  > - 函数体被拆成状态机（when(state)），每次跨挂起点恢复从对应分支继续——所以**局部变量要能跨挂起存活**，编译器会把它们放进状态对象。
  >
  > **dispatch**：恢复（resumeWith）经由 Continuation.intercepted()（CoroutineContext 里的 ContinuationInterceptor，如 Dispatchers.Main 的 Handler）切到目标线程执行状态机下一段。
  >
  > 面试追问："suspend 会阻塞线程吗？"不会——它只是让出 Continuation，线程去跑别的协程/消息；阻塞的是"逻辑"不是线程。能答出 CPS/状态机而不是"挂起就是暂停线程"，即达到源码级。

* CoroutineScope、Job、SupervisorJob 与结构化并发

  > **结构化并发**：子协程必须属于某个父 Job；父取消→子全取消；子失败→默认**父失败、兄弟全取消**（fail-fast）。
  >
  > **SupervisorJob**：子失败不传染父与兄弟，各自独立失败——UI 中一个子任务崩不影响整个页面协程（配合 CoroutineExceptionHandler 捕获）。
  >
  > **viewModelScope**：ViewModel 内置 SupervisorJob + Main.immediate，onCleared 自动 cancel——替代 AsyncTask 的生命周期安全。
  > **lifecycleScope**：Activity/Fragment 生命周期 scope，destroy 自动取消。
  >
  > 陷阱：GlobalScope 无结构化、不随页面取消 → 泄漏与"页面关了任务还在跑"；自建 scope 必须显式 cancel，否则测试泄漏。

* 异常传播与 CoroutineExceptionHandler 的正确用法

  > - launch：异常默认**立即传播**给父 Job，取消整个 scope（除非 SupervisorJob）。
  > - async：异常**不立即传播**，在 await 时抛出；不 await 会变成未捕获异常（线上崩溃源）。
  > - try/catch 只能包住 **非挂起边界内的同步异常**；跨挂起的异常走协程机制，需 handler 或 catch 在调用 await/join 的边界。
  > - handler 只能安装在"根协程"（scope.launch 的最外层），子协程安装无效（异常先上抛给父）。
  >
  > 推荐模式：UI 层 `scope.launch { try { ... } catch (e: Throwable) { 展示错误 } }` 或统一 handler + 全局崩溃上报挂钩。

* Flow 的冷流特性、collect 与线程切换（与 RxJava 对照前的铺垫）

  > Flow 是**冷流**：没有 collect 前代码不跑；每次 collect 重新执行生产逻辑（除非 sharedIn/stateIn 共享）。
  >
  > `flow { emit(...) }` 在 collect 的线程执行；`flowOn(Dispatchers.IO)` 改变**上游**生产线程（只影响上游，下游仍在 collect 线程）——与 RxJava subscribeOn 类似但更精确。
  >
  > StateFlow/SharedFlow 是**热流**：总有值/被共享，配置变更后 ViewModel 暴露 StateFlow 是当前 MVVM 标配（替代 LiveData 的 lifecycle 自动感知需配合 repeatOnLifecycle 收集）。

* RxJava 核心概念速览（可选但常被问）：观察者扩展、线程模型、背压

  > **核心**：被观察者 Observable/Flowable + 观察者 Observer + 订阅 Subscribe + 操作符 map/filter/flatMap + 调度器 Scheduler。
  >
  > **调度器**：Schedulers.io()（IO 线程池，弹性）、computation()（CPU 核数）、newThread()、mainThread()（Android 主线程 Handler）——对应协程 Dispatchers.IO/Default/Main。
  >
  > **背压**：Observable 无背压（下游跟不上就丢/OOM）；Flowable 有背压策略（BUFFER/DROP/LATEST/ERROR）。
  >
  > **与协程对比**：RxJava 强在操作符生态与背压成熟度；协程胜在结构化并发、生命周期自动取消、与 Kotlin 语法融合。迁移：Flow + operators 大部分可覆盖；复杂事件流可仍用 Rx。
  >
  > 面试陷阱：RxJava 忘记 subscribe 返回的 Disposable 在 onDestroy dispose → 同 Handler 泄漏；CompositeDisposable 统一管理是模板代码。

* 面试官视角

  > **考察点**：协程是否理解到编译产物（CPS/状态机）与异常传播模型，而非只会 launch/withContext 皮毛。
  >
  > **加分**：COROUTINE_SUSPENDED 协议、SupervisorJob 边界、GlobalScope 之害、Flow 冷/热与 repeatOnLifecycle、Rx 背压概念。
  >
  > **减分**：说"suspend 就是切线程"；把 try/catch 当能捕获所有协程异常；在 UI scope 里用 GlobalScope。
