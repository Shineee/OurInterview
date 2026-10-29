# Android 框架层笔试题

* 简述 Handler 消息机制的四个角色及完整工作流程（要求讲到消息入队、出队与同步屏障）。

  > **角色**
  > Handler：发送消息（sendMessage/post）与处理消息（handleMessage/run），同时是消息的 target——每条 Message 持有 msg.target 指回发送它的 Handler。
  >
  > MessageQueue：按 when 排序的单链表消息队列；现代实现中队列本体在 native 层（Android 8.0 起 enqueue/next 下沉到 native MessageQueue，减少 Java/native 往返），Java 层仍保留同步锁与 IdleHandler 逻辑。
  >
  > Looper：通过 Looper.prepare 为线程绑定唯一实例（ThreadLocal 保证），loop() 是死循环调用 next() 取消息；主线程 Looper 由 ActivityThread.main 中 prepareMainLooper 创建，quit 会抛异常。
  >
  > Message：消息载体， obtain() 从全局同步池复用，避免高频 GC。
  >
  > **流程**
  > 1) sendMessage 最终走到 MessageQueue.enqueueMessage：按 when 插入链表；若队头就是本消息则唤醒 nativePollOnce（写 eventfd/管道），打断 epoll 等待。
  > 2) Looper.loop 中 MessageQueue.next：有到期消息则返回；无消息则 nativePollOnce 阻塞（idle 时线程挂起，不占 CPU——这回答了"死循环为什么不卡死"）。
  > 3) 取到消息后按 msg.target 回调 dispatchMessage，优先级：msg.callback（post 的 Runnable）> mCallback > handleMessage。
  >
  > **加分点——同步屏障与异步消息**：ViewRootImpl.scheduleTraversals 会 postSyncBarrier 并发送异步消息（setAsynchronous(true)），使绘制相关的 Message 跳过队列中的普通同步消息优先执行，保证 UI 不被积压的普通消息阻塞；绘制完成后 removeSyncBarrier。
  >
  > 【追问】postDelayed 的延迟是在哪一层实现的？——入队时记录 when，next() 中计算 delay，native 层 epoll_wait(timeout) 等待，精度受系统负载影响。

* Handler 使用不当为什么会造成内存泄漏？WeakReference 方案为什么"治标不治本"？

  > **引用链（GC 不可回收的完整路径）**
  > GC Root（主线程 Looper / MessageQueue）-> 挂起的 Message -> msg.target（Handler）-> 非静态内部类 Handler -> 外部 Activity。
  > 只要延迟消息还在队列中未被移除，整条链可达，Activity 无法回收——泄漏的不是 Handler，是整棵以 Activity 为根的视图树（View 树持有 Context，代价极大）。
  >
  > **标准解法**
  > 静态内部类 + WeakReference<Activity>：打破 Handler->Activity 的强引用；onDestroy 中 handler.removeCallbacksAndMessages(null) 清空队列。
  >
  > **WeakReference 的局限（面试区分度）**
  > 1) 弱引用只保证"不再阻止回收"，不保证"及时回收"——若 Activity 已 finish 但 GC 未运行，handleMessage 仍可能操作已销毁的界面，产生逻辑 bug 而非泄漏；
  > 2) 若 handleMessage 中仍通过 weakRef.get() 拿到引用并做耗时操作，对象存活期间照样卡主线程；
  > 3) 真正的根治是生命周期感知：协程 viewModelScope/lifecycleScope 自动取消，或 LiveData.postValue——取消时机由框架按生命周期保证，不依赖人肉 remove。
  >
  > 【追问】ThreadLocal 在 Handler 体系里出现几次？——Looper 的绑定（myLooper）、以及部分框架的线程隔离设计，本质都是"线程私有变量"。

* 简述 View 的事件分发机制（三个核心方法），并说明滑动冲突的典型解决思路。

  > **分发链**：Activity.dispatchTouchEvent -> DecorView -> ViewGroup.dispatchTouchEvent -> onInterceptTouchEvent -> 子 View -> 不消费则冒泡回父 onTouchEvent。
  >
  > dispatchTouchEvent：决定传给子 View 还是自己处理；
  > onInterceptTouchEvent：ViewGroup 独有，决定是否拦截（默认 false）；一旦拦截，后续同手势事件不再询问 onInterceptTouchEvent，直接走自己的 onTouchEvent；
  > onTouchEvent：返回 true 表示消费；mFirstTouchTarget 记录首个消费的子 View，后续事件按该链表分发。
  >
  > **滑动冲突两类解法**
  > 外部拦截法：父容器 onInterceptTouchEvent 根据业务条件（滑动方向、起始坐标差）决定拦截；
  > 内部拦截法：子 View 在 dispatchTouchEvent 中通过 requestDisallowInterceptTouchEvent(false) + 父容器配合拦截，把决定权交给子 View。
  >
  > 【追问】事件流中 ACTION_CANCEL 何时出现？——父容器中途开始拦截、或手势被系统打断（如新触摸点接管）时，给子 View 发 CANCEL 并终止其处理。

* 什么是 ANR？请给出从"现象"到"根因堆栈"的完整排查路径。

  > **定义与阈值**：主线程在期限内未完成系统调度的工作。输入事件 5s；Service 前台约 20s / 后台约 200s（版本略有差异）；广播接收 10s；ContentProvider 超时。本质是主线程消息循环被单条消息长时间占用。
  >
  > **排查路径（资深答法）**
  > 1) 现场：log 中 am_anr 关键字，拿到 pid 与原因分类（Input dispatching timed out / Broadcast / Service）。
  > 2) 堆栈：/data/anr/traces.txt（或 Android 8+ /data/anr/anr_*），看 "main" 线程当前 stack——是锁等待（waiting to lock）、IO（read/stat）、Binder 同步调用对端无响应，还是死循环。
  > 3) 锁与 Binder：traces 中标明 held by / waiting on；Binder 等待则需抓对端进程（system_server 或服务进程）的 trace，判断是否对端主线程被占。
  > 4) 系统侧：systrace / Perfetto 查看主线程 Message 处理时长、Binder 事务耗时、IO 等待（D 状态）、CPU 被抢占（低优先级被饿死）。
  > 5) 高频复现场景：StrictMode 抓主线程磁盘/网络 IO，结合自埋点定位具体业务消息。
  >
  > **高频根因清单**：主线程网络/磁盘 IO、SharedPreferences commit 同步写与首次加载全量 XML、锁竞争（如主线程等子线程持有的 DB 锁）、同步 Binder 对端阻塞、大量反射/类加载、WebView 初始化、主线程被 GC 频繁打断（分配速率过高）。
  >
  > 【追问】onCreate 里 getSharedPreferences 第一次读为什么会 ANR？——首次加载全量 XML 到内存且同步阻塞；大 SP 文件 + 主线程并发写锁会显著放大。

* 一张 1080x1920 的 ARGB_8888 Bitmap 占用多少内存？若要加载 4000x3000 的原图到 ImageView，如何避免 OOM？

  > 1080*1920*4 ≈ 7.9MB；公式：宽*高*每像素字节数（ARGB_8888=4B，RGB_565=2B，HARDWARE 自 8.0 起存 native，不占 Java 堆但受总内存限制）。
  >
  > 4000*3000*4 ≈ 45.8MB——单张就可能逼近旧设备堆上限。避免手段：
  > 1) inJustDecodeBounds=true 先拿尺寸不分配像素；结合目标 ImageView 尺寸算 inSampleSize（2 的幂）采样；
  > 2) inPreferredConfig 降采样格式；大图用 RGB_565（无透明时）；
  > 3) inBitmap 复用已有内存块（Android 4.4 后允许尺寸不严格相等的复用）；
  > 4) 列表场景用 Glide/Coil 等库自动做 downsample + 生命周期暂停 + 内存/磁盘缓存，不要手写 Bitmap 传递链。
  >
  > 【追问】Android 8.0 后 Bitmap 像素放哪了？——native 内存（NativeAllocationRegistry 注册并联动 GC），堆内只留少量 Java 壳，因此"堆没满也 OOM"要查 native。

* 列举常见的 Android 线程间通信方式，并说明"UI 线程与后台线程"场景下的推荐组合。

  > Handler/Looper/MessageQueue：最基础、最常考；Messenger：对 Binder 的轻封装，适合简单跨进程；AIDL/Binder：跨进程方法调用；LiveData/StateFlow：生命周期感知的主线程回调；RxJava/协程：声明式线程切换；共享内存 + 锁/wait-notify：底层手段。
  >
  > **推荐组合（资深视角）**：协程 viewModelScope + Room/ Retrofit suspend + StateFlow 收集更新 UI——取消由生命周期保证，无 Handler 泄漏面，无 AsyncTask 串行坑。存量代码维护才需要手写 Handler 细节。

* 【系统设计简答】设计一个图片加载库，需要哪些核心模块？各自职责是什么？

  > 1) API 层：链式调用（with/load/into），与生命周期组件绑定（感知 Activity/Fragment 防泄漏）；
  > 2) 请求管理：任务去重（同一 URL 多 View 请求合并）、暂停/恢复（列表滑动时暂停 offscreen 请求）；
  > 3) 缓存层：活跃资源 -> 内存 LRU -> 磁盘 Resource（解码后）/Data（原始）四级，key 由 URL + 变换 + 尺寸构成；
  > 4) 解码与变换：inSampleSize 采样、圆角/模糊等 Transformation、目标尺寸（Target 尺寸驱动 downsample）；
  > 5) 网络层：复用 OkHttp 连接池，支持鉴权头与重试；
  > 6) 回调与容错：占位图、错误图、渐进式加载、取消时回调语义。
  >
  > 一句话考察点：是否理解"缓存 key 设计"与"生命周期 + 请求去重"才是图片库的核心难点，而不是会调 API。

* 【排查题】列表页快速滑动时主线程周期性掉帧（systrace 显示 main 有 10ms+ 的间隙），自定义 Adapter 中每次都 inflate 新 View，如何定位与优化？

  > 定位：Perfetto/systrace 看掉帧处是否 inflate/bindView；Android Studio Profiler 看 allocations 速率与 GC 频率；确认 RecyclerView 是否走了缓存（log 回收复用路径）。
  >
  > 优化优先级：1) 正确使用 ViewHolder 复用（减少 findViewById，用 viewBinding 缓存）；2) prefetch（GapWorker）对齐，异步预 inflate（AsyncLayoutInflater）；3) 减少 onBindViewHolder 中的计算与图片直接解码（交给图片库）；4) 布局层级扁平化（ConstraintLayout）、避免过度绘制；5) 数据侧 diff 化（DiffUtil）减少整段 notify。
  >
  > 考察点：能否沿"现象 -> 布局/分配/GC/复用"四条证据链定位，而不是背"用 ViewHolder"。

---

* View 的 measure、layout、draw 三个过程简述。

  > measure：自顶向下递归计算宽高，MeasureSpec 由父容器要求和自身 LayoutParams 共同决定；自定义 View 需重写 onMeasure 并处理 wrap_content。
  >
  > layout：父容器确定子 View 在父容器中的位置（left/top/right/bottom）。
  >
  > draw：绘制背景 -> onDraw 内容 -> 绘制子 View（dispatchDraw）-> 绘制前景/滚动条等装饰。

* 为什么自定义 View 时 wrap_content 经常表现为 match_parent？

  > 因为默认 View 的 onMeasure 中，EXACTLY 模式下直接取 specSize，AT_MOST 模式（wrap_content 对应）也直接用了 specSize，没有根据内容计算。
  >
  > 解决：在 onMeasure 中对 AT_MOST 模式自行测量内容尺寸，并用 setMeasuredDimension 保存。

* Activity、View、Window、ViewRootImpl 的关系。

  > Activity 持有一个 Window（PhoneWindow）；PhoneWindow 的 DecorView 内容区域承载 View 树。
  >
  > ViewRootImpl 是 View 树与 WindowManagerService 之间的纽带：负责 View 树的绘制、事件分发入口（ViewRootImpl 接收输入事件后传给 DecorView）、measure/layout/draw 的实际驱动者。
  >
  > ViewRootImpl 在 Activity 的 handleResumeActivity 中 attach 到 WindowManager。

* 主线程和 UI 线程的关系是什么？Looper.loop() 是死循环，为什么不会卡死应用？

  > ActivityThread.main 中通过 Looper.prepareMainLooper 建立主线程 Looper，UI 更新必须在持有该 Looper 的线程执行。
  >
  > loop() 的无限循环本质是消息驱动：无消息时 MessageQueue.next 在 native 层 epoll 等待，不占用 CPU；有消息时被唤醒处理，这就是"没有消息处理时主线程阻塞但不 ANR"的原因——ANR 看的是消息能否及时被处理，而不是循环本身。

* 简述 synchronized、volatile 在 Android 并发中的作用。

  > volatile：保证可见性与禁止指令重排，不保证原子性；典型用于标志位、DCL 单例的 instance 字段。
  >
  > synchronized：保证互斥 + 可见性；Android 中还需注意线程阻塞导致的优先级反转/ANR 风险，官方推荐更轻量的 ReentrantLock + Condition（AsyncTask/Handler 中可见此类用法）。

* 一个 App 启动一个新的 Activity 大致经过哪些跨进程调用？

  > App 进程通过 startActivity 向 AMS（system_server 进程）发起请求（Binder 调用）。
  >
  > AMS 检查权限、处理启动栈、决定是否新建进程；若目标进程不存在，通过 Zygote fork 新进程。
  >
  > 新进程 ActivityThread.main 执行，向 AMS 注册 ApplicationThread，再由 AMS 回调 scheduleLaunchActivity，App 进程才真正走 Activity.onCreate。
