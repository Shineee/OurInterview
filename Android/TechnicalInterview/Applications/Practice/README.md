# 实践题

* A 应用已发布，用户来电说网上下载了一个 A 应用，无法确定是否官方版本，如何验证？考察什么、底层原理是什么

  > **方案**：给用户官网下载地址，覆盖安装——成功则是官方版本（签名一致），失败（INSTALL_FAILED_UPDATE_INCOMPATIBLE）则是盗版。
  >
  > **底层原理**：Android 应用签名基于 APK 内 `META-INF/*.RSA` 中的证书 + MANIFEST.MF 摘要。PackageManagerService 在 install/update 时校验新旧包签名（v1/v2/v3/v4 签名方案），签名不一致直接拒绝，防止提权与覆盖攻击。每个开发者的 keystore 私钥唯一 → 签名唯一 → 天然的"官方性"证明。
  >
  > **追问链**：
  > - v1/v2/v3 区别：v1 JAR 签名（逐文件摘要，易受 Janus 漏洞）；v2 整个 APK 字节块签名（更快更严）；v3 支持密钥轮换（proof-of-rotation）。v4 增量安装用。
  > - 应用市场分发的"重签名包"：某些市场会用自己证书重签，导致无法覆盖官方包——这也是"同一 App 多渠道签名不一致"的坑。
  > - 更严格的企业方案：比对包名 + 签名指纹（`apksigner verify --print-certs`）+ 服务器下发官方签名白名单校验。
  >
  > **面试官视角**：此题表面考"怎么验证"，实则考你对 Android 签名与安装链路（PMS）的理解。加分是讲清 v1/v2/v3 与提权防御；减分是只答"看图标/看大小"。

* 一个线上 App 频繁 OOM，如何系统化定位？（内存泄漏排查完整方法论）

  > **第一步：确认现象与分层**
  > - 区分 Java 堆 OOM（OutOfMemoryError: Java heap space）与 native OOM（Failed to allocate ... native allocation）。两者工具链不同。
  > - 看 `dumpsys meminfo <pkg>`：对比 Java Heap / Native Heap / Graphics / TOTAL 随页面操作的增长趋势；泄漏通常是"打开某页面 → 返回 → heap 不回落"。
  >
  > **第二步：归因到对象**
  > - Debug.dumpHprofData 或 Android Studio Profiler 抓 heap dump，按 Retained Size 排序找异常大对象。
  > - LeakCanary（生产可用精简版）监控 Activity/Fragment/ViewModel 销毁后是否被 GC Root 引用：自动 dump + Trace 到引用链。
  > - 常见引用链模式：
  >   1. 非静态内部类 Handler/Runnable → MessageQueue → 外部 Activity。
  >   2. 单例/静态 Map 持有 Context。
  >   3. 未注销的监听：SensorManager.registerListener / LocationManager / EventBus / BroadcastReceiver / ContentObserver。
  >   4. WebView 泄漏：Activity 持 WebView，WebView 内部持有 Activity；需先 destroy WebView 再销毁 Activity，或 WebView 放独立进程。
  >   5. 属性动画未取消：ValueAnimator 持 Target（View→Activity），无限循环动画在后台继续跑。
  >   6. 线程池/回调跨生命周期持有弱引用缺失。
  >
  > **第三步：治理与预防**
  > - 架构：Context 只用 Application 级做长生命周期持有；组件销毁链路统一（onDestroy 模板：注销所有注册）。
  > - CI 卡口：LeakCanary debug 全量 + 关键页面 monkey 跑一遍看 leak 报告。
  > - 线上：接入内存监控 SDK（采样 hprof 上报 + 私有化解析），按泄漏对象聚合告警。
  >
  > **面试官视角**：考察"现象→指标→heap→引用链→治理"的闭环，而非只说"用 LeakCanary"。加分提多进程 meminfo、native 内存、属性动画与 WebView 这些非典型泄漏；减分只会背"静态变量持有 Context 会泄漏"。

* 为什么单例持有 Activity Context 会泄漏？持有 Application Context 就一定安全吗

  > 单例生命周期≈进程，Activity destroy 后堆上仍有从 GC Root（单例静态字段）出发的引用链 → 无法回收 → 泄漏。
  >
  > Application Context 生命周期与进程一致，持有它不会造成"Activity 泄漏"，但注意：
  > 1. 若单例里缓存了与 Activity 绑定的对象（如 Dialog、 inflater 的 ContextThemeWrapper、WebView），仍会间接持有 Activity。
  > 2. 用 Application Context startActivity 必须带 FLAG_ACTIVITY_NEW_TASK，否则抛异常——这是"安全但有使用约束"。
  > 3. 真正的"泄漏"定义是"不再需要的对象被 GC Root 可达"；Application Context 本身不该被"回收"，所以不算泄漏，但无节制地往全局单例塞数据是内存膨胀（非泄漏）的来源。

* 内存抖动（Memory Churn）与内存泄漏的区别，如何用 Profiler 区分

  > **泄漏**：对象在不该存活时仍被引用，堆**单调增长**，最终可能 OOM；heap dump 看到异常 Retained Size 的引用链。
  >
  > **抖动**：短时间大量临时对象分配/回收，GC 频繁（GC_CONCURRENT/GC_FOR_ALLOC），造成**掉帧**（主线程分配触发 GC 暂停）；表现为锯齿状内存曲线、Allocation Tracking 里同一调用栈反复分配。
  >
  > 常见抖动源：onDraw 里 new Paint/Rect、onScroll 频繁 create 字符串、循环里装箱（int→Integer）、onLayout 反复 requestLayout 触发重复测量分配。
  >
  > 治理：对象池/复用（Paint 预创建）、避免自动装箱热点、RecyclerView.setItemViewCacheSize、用 Baseline Profile 减少解释执行期分配。

* 面试官视角（本页总览）

  > **考察点**：内存问题的完整工程能力——从 dumpsys/Profiler 工具到引用链推理，再到架构预防与线上监控，而不是单点背诵。
  >
  > **加分**：区分 Java/native、抖动 vs 泄漏、WebView/动画/回调等非经典案例、CI 与线上治理闭环。
  >
  > **减分**：只会"装个 LeakCanary"；把"内存高"直接当泄漏；不知道多进程下 meminfo 要分进程看。

---

* 冷启动优化的体系化方法论（不是罗列技巧，而是可度量的工程）

  > **定义与度量**：冷启动 = 进程 fork → Application.onCreate → 首 Activity 首帧上屏（reportFullyDrawn 或首帧 vsync）。指标：`adb shell am start -W` 的 TotalTime/WaitTime；线上用 App StartUp 库/自埋点量"点击图标 → 首帧可交互"分位数（P50/P90/P99），分设备档位。
  >
  > **白盒定位（按链路分段）**：
  > 1. 进程 fork ~ attachApplication：系统开销，应用难优化；关注是否被"后台拉起"拖累设备整体。
  > 2. Application.onCreate：**最大可优化区**。用 Trace.beginSection / Systrace 找出同步初始化的三方 SDK（推送、统计、加固、WebView 预热）。策略：异步线程池 init（注意线程安全与依赖顺序）、按需延迟（首个用到时再 init）、或用 App Startup 库管理初始化器依赖图。
  > 3. 首 Activity.onCreate/onResume：布局复杂度（深度/测量次数）、RecyclerView 首屏数据是否同步请求、是否主线程 IO（SP 首次 get、so 加载）。
  > 4. 首帧：是否被 GC、是否多次 requestLayout、Starting Window 是否被过早移除（窗口背景与真实背景不一致的"闪屏"）。
  >
  > **黑盒基线**：Systrace/Perfetto 抓启动段，看主线程 binder、GC、inflate、Class verify 占比；Macrobenchmark 库自动化跑冷启动分位。
  >
  > **结构性手段**：Baseline Profile（AOT 关键路径，减少解释/JIT）、启动窗口与真实布局视觉一致（避免"白→内容"跳变）、多进程初始化隔离、页面级懒加载/预加载策略。
  >
  > **陷阱**：盲目"全部异步化"引发线程竞争与初始化顺序 bug；延迟 init 把耗时挪到首次使用时造成"使用卡顿"（把冷启动债挪到交互处）。正确做法是"按依赖分层 + 可度量前后对比 + 灰度看大盘分位"。
  >
  > **面试官视角**：考察是否把启动优化当"指标+分段+治理"工程，而不是背"异步初始化、懒加载、减少布局层级"三板斧。加分提到 Macrobenchmark/Baseline Profile、Starting Window 一致性、异步化的副作用治理。

* UI 卡顿（掉帧）的归因路径与工具链

  > **现象指标**：FPS/掉帧率、Janky frames（dumpsys gfxinfo <pkg> framestats）、线上用 FrameMetrics / JankStats（Jetpack）采集慢帧分布。
  >
  > **归因分层（自顶向下）**：
  > 1. **主线程耗时**：Message 处理超一帧（16.6ms/60Hz、8.3ms/120Hz）。用 Profiler/Systrace 找超长 Message/Callback：常见 inflate、序列化、SP commit、正则、大循环。
  > 2. **锁竞争**：主线程等锁（monitor contention），Systrace 看到 blocked。常见：主线程调用持锁的单例、Binder 线程与主线程抢锁。
  > 3. **GC 暂停**：Allocation Tracking + heap 曲线锯齿；STW 暂停虽短，高频分配叠加会掉帧。
  > 4. **布局层级/测量**：过度嵌套、wrap_content 多次测量、requestLayout 风暴（如 ScrollView 嵌套 RecyclerView 滚动时互相 requestLayout）。用 Layout Inspector + `dumpsys gfxinfo` + GPU 过度绘制（开发者选项"调试 GPU 过度绘制"）。
  > 5. **过度绘制（Overdraw）**：同一像素多层绘制，GPU 填充率压力大。色块图：蓝 1x 可接受，红 4x+ 需移除背景。
  > 6. **绘制管线外**：主线程被 Binder 阻塞（如 AMS 查询）、IO page fault、so 加载。
  >
  > **排查工具组合拳**：Perfetto（含 atrace 的 sched/binder/gfx）→ 定位慢帧所在 Message → Android Studio Profiler 看方法栈 → Layout Inspector/GPU rendering bars 确认绘制侧问题。
  >
  > **面试官视角**：考察"慢帧→主线程哪段→什么类型（CPU/锁/GC/布局/GPU）"的推理链。加分提到 VSYNC 与 Choreographer、三缓冲、JankStats 线上化；减分只会"用 Systrace 看一眼"。

* RecyclerView 滚动卡顿的常见根因与对策

  > 1. **onBindViewHolder 过重**：格式化日期、复杂测量、图片同步加载。→ 异步预处理数据、DiffUtil 细粒度更新、图片异步+占位。
  > 2. **onCreateViewHolder 频繁 inflate**：预创建 ViewHolder 池（setItemViewCacheSize、共享 RecycledViewPool 跨列表）。
  > 3. **嵌套滚动与 requestLayout**：横向 RV 套在纵向 RV item 内，滚动时相互测量。→ 固定 item 高度、用 concatAdapter、避免嵌套 RV 内再用 wrap_content。
  > 4. **图片未复用/未取消**：快速滑动时旧请求覆盖新 ViewHolder（错图）。→ 按 tag 绑定、Glide 自动取消、或 ListAdapter + 异步提交。
  > 5. **ItemAnimator 开销**：默认 change animation 会闪烁且重测。列表静态可 setItemAnimator(null)。
  > 6. **预测式返回/共享元素**转场在复杂列表上开销大，按需开启。
  >
  > 追问：**预取（Prefetch）**——GapWorker 在滚动空闲期提前 inflate/bind 即将进入屏幕的 item（RV 有 prefetchIndex），这就是"滑动更顺"的秘密；面试提到 GapWorker 说明真读过 RV 源码。

* ANR 的排查路径（从 dropbox 到 trace）

  > **现场**：系统弹"应用无响应"时生成 `/data/anr/traces.txt`（或 dropbox 的 data_app_anr），里面有主线程与所有线程的 Java 栈 + 部分 native 栈。
  >
  > **步骤**：
  > 1. 抓 traces：`adb pull /data/anr/`（或 bugreport）。
  > 2. 看主线程状态：RUNNABLE（死循环/CPU 忙）、WAITING/TIMED_WAITING（等锁，看 holding 那把锁的线程）、Native（Binder/IO）。
  > 3. Binder 阻塞：主线程栈顶在 Binder.transact，对端是 system_server 慢调用（如 PMS、AMS）→ 检查是否主线程频繁跨进程。
  > 4. 锁：找 monitor 字段指向的锁被哪个线程长期持有。
  > 5. 结合 CPU 使用（traces 顶部或 bugreport 的 cpuinfo）区分"CPU 忙导致处理不过来"与"IO/锁等待"。
  >
  > 线上治理：WatchDog 监控主线程心跳（Handler post 延迟检测）、ANR 上报（捕获 SIGQUIT 时机或 dropbox 读取权限受限时用系统 ApplicationExitInfo，Android 11+ 可读最近退出原因含 ANR）。
  >
  > **面试官视角**：考察是否处理过真实 ANR 文件，能区分死锁/忙等/Binder 三大类，而不是只说"主线程别做耗时"。

* 面试官视角（本页总览）

  > **考察点**：性能优化是否"先度量后优化、分段归因、有治理闭环"，反对没有基线的玄学优化。
  >
  > **加分**：Macrobenchmark/JankStats/Baseline Profile 现代工具链；提到异步初始化的副作用；GapWorker 预取；ApplicationExitInfo。
  >
  > **减分**：只会列"避免主线程 IO、减少布局层级、用缓存"；说不出指标怎么采、优化前后怎么对比。

---

* 包体积优化的分层策略与度量

  > **度量**：Android Studio APK Analyzer 看各部分占比（dex/so/res/assets/lib）；线上关注下载转化与低存储设备安装失败率。
  >
  > **资源层**：
  > - 移除无用资源（ShrinkResources + Lint 的 UnusedResources）。
  > - 图片：WebP（有损/无损）、矢量图替代小 PNG、大图放 xxhdpi 基准 + 动态下发、移除多语言不需要的 locale。
  > - 资源混淆（AndResGuard/R8 resource shrinking）压缩路径字符串。
  >
  > **代码层**：
  > - R8/ProGuard：代码压缩 + 混淆 + 无用代码删除；注意 keep 规则与反射/序列化的兼容（-keeprules 最小化）。
  > - 移除无用依赖与功能模块（组件化后按 App 分发不同 feature 模块）。
  >
  > **SO 层**：
  > - abiFilters 只打主流 ABI（arm64-v8a 为主，按渠道/设备分发 armeabi-v7a）。
  > - strip 调试符号、压缩 so（upx 有兼容风险）、按需加载（ReLinker / System.loadLibrary 延迟）。
  >
  > **动态化与架构层**：插件化/动态下发资源（合规风险）、feature on-demand（Play Feature Delivery / 国内自研）。
  >
  > **面试官视角**：考察分层治理与工具链，而非只会"开 minifyEnabled"。加分提到 R8 规则维护成本、多 ABI 分发、AndResGuard；减分不知道 APK Analyzer、把"压缩图片"当唯一手段。

* 线上稳定性治理：崩溃与卡顿的监控体系怎么搭

  > **崩溃**：
  > - Java：Thread.setDefaultUncaughtExceptionHandler，配合 Breakpad/Crashpad 抓 native。
  > - Native：signal handler + minidump（Breakpad/Crashpad），符号化在服务端（保留带符号 so）。
  > - 集成方：Bugly/Firebase Crashlytics 类比——聚合堆栈、按版本/设备/页面维度聚类、新版本崩溃率突增告警。
  > - 崩溃率指标：Java 崩溃率、Native 崩溃率、整体用户影响面（崩溃用户数/DAU），分版本灰度看。
  >
  > **ANR**：如前页（traces + WatchDog + ApplicationExitInfo）。线上 ANR 率往往比崩溃更伤体验。
  >
  > **卡顿/慢帧**：JankStats/自研主线程监控（编译插字节码插桩对关键生命周期与页面 onCreate/onResume 打点），结合堆栈采样（profilo 类似方案：周期采主线程栈聚合）。
  >
  > **治理流程**：新版本灰度分阶段放量 → 崩溃率/ANR 率大盘对比基线 → 超阈值自动回滚/暂停放量 → 长尾 crash 按堆栈 owner 分派修复。
  >
  > **面试官视角**：考察"采集→聚合→告警→回滚"的工程闭环与指标定义（分位数、影响面），而非只说"接个 Bugly"。

* 启动即崩、白屏、黑屏等典型现场的快速归因

  > **启动即崩（Java）**：看 Application.onCreate / ContentProvider init（很多 SDK 静默在 provider 里初始化）。三方 SDK 在主线程读网/读盘未做异常保护是重灾区。
  >
  > **启动即崩（Native）**：so 加载（System.loadLibrary）ABI 不匹配、符号缺失、或多线程 fork 后在子线程用未初始化的 native 状态。
  >
  > **白屏久**：Starting Window 一直在，应用首帧未上屏 → 冷启动链路过长（前页方法论）；或主线程卡在 Application.onCreate 死锁。
  >
  > **黑屏/花屏**：SurfaceFlinger 合成异常、硬件加速兼容问题（特定 GPU 上的 RenderNode/Path 故障）、TextureView/SurfaceView 生命周期错配（Surface 销毁后还往里画）。
  >
  > 归因工具：logcat + dropbox + tombstone（native）+ Perfetto 看启动段 + `dumpsys SurfaceFlinger`。

* 大型 App 的进程架构设计考量

  > **为什么多进程**：
  > 1. 隔离崩溃：某模块 native 崩溃不拖垮主进程。
  > 2. 隔离内存：WebView/推送/下载放独立进程，主进程堆压力小，LMK 杀子进程不杀主进程。
  > 3. 保活/优先级：前台 Service 进程独立，后台任务存活更久。
  >
  > **代价**：
  > - 静态变量/单例失效（每进程一份），需跨进程通信（AIDL/ContentProvider/文件+监听）。
  > - Application.onCreate 每进程都跑，需按进程名分流初始化（"主进程才 init 推送"）。
  > - 调试与监控复杂度上升（每个进程单独 meminfo/trace）。
  >
  > **常见布局**：主进程（UI）、:push、:webview、:download、:plugin。用 `Application.getProcessName()` 或自建 ProcessUtils 分流。
  >
  > **面试官视角**：考察"要不要多进程"的 trade-off 思维，而非"多进程高级所以用"。加分提到 Application 分进程初始化与跨进程单例失效。

* 面试官视角（本页总览）

  > **考察点**：工程化视野——体积、稳定性、进程架构都是"体系"而非单点。
  >
  > **加分**：指标定义、灰度回滚闭环、R8/abi 分发细节、进程隔离的 trade-off。
  >
  > **减分**：只罗列名词没有因果；把动态下发当万能（合规/性能代价）；不知道 ContentProvider 在 SDK 初始化中的作用。
