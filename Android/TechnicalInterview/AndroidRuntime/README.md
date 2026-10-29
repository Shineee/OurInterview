# Android Runtime 面试题

* Dalvik 与 ART 的本质区别（为什么 4.4 后全面转向 ART）

  > **Dalvik**：早期 Android 虚拟机，基于寄存器，使用 **JIT（Just-In-Time）**——运行时把热点字节码编译成本机码，下次执行才快；每次冷启动都要解释+重新热点编译，且 JIT 结果不持久。
  >
  > **ART（Android 5.0+）**：默认 **AOT（Ahead-Of-Time）**——安装/系统升级时（dex2oat）把 dex 编译成机器码存为 **.oat**，运行直接执行，启动与运行更快、更省电。5.0 纯 AOT 导致安装慢/存储大，后引入**混合模式**：安装时 Profile-Guided（记录热点）+ 空闲/充电时 dex2oat 补全 + 保留 JIT 处理新热点（解释器兜底）。
  >
  > 影响面试点：安装体积（oat 占空间）、首次启动性能、Baseline Profile 原理（给 ART 提供热点方法清单，让 PGO 更准）。

* dex、odex、vdex、art、oat 文件分别是什么

  > - **dex**：Dalvik 可执行字节码（多 class 合并，减重复冗余），打包在 APK 的 classes*.dex。
  > - **odex**：优化后的 dex（早期 Dalvik 时代，从 APK 解出优化，加快加载）。
  > - **oat**：ART 的 ELF 文件，包含编译后的机器码 + 原始 dex（或引用）。
  > - **vdex**：验证过的 dex 容器（避免重复 verify），与 oat 配合（Android 8+）。
  > - **.art**：堆镜像（boot image），预初始化的类/对象，Zygote 共享的基础。
  >
  > 面试串联：安装时 dex2oat 读 dex → 产 oat/vdex；Zygote 加载 boot.art/boot.oat 预热；应用 fork 后共享这些只读页（COW）。

* 应用进程如何从 Zygote fork 出来（启动链路 runtime 视角）

  > system_server 需要新进程时，通过 socket 通知 **Zygote**（预加载 framework 类/资源、已开 ART 虚拟机）执行 fork()——**在 Zygote 主线程 fork 保证安全**（单线程无锁竞争）。
  >
  > fork 后子进程：
  > 1. 继承预加载的类与堆（COW，写时才复制）→ 冷启动只需加载自身 dex。
  > 2. 关闭继承的 socket，设置进程名/uid/安全上下文。
  > 3. 调 ActivityThread.main → attach → 走应用初始化（链接 Activity 第5页）。
  >
  > 这就是"Android 多 App 共享 Zygote 预加载"的内存与速度优势；也解释了多开/分身类应用的技术起点（多 fork 或改包名）。

* 进程优先级（oom_adj）与 LMK 回收模型

  > AMS 为每个进程计算 oom_adj（前台 0 量级，缓存进程 900+），内核 **lowmemorykiller** 在内存压力（PSI/水位）时从高 adj 杀起；Android 10+ 有更细的 App Standby Bucket 与 cached 进程管理。
  >
  > 面试可链接：前台 Service / 可见 Activity 降低被杀概率；"保活"黑产对抗的就是这套模型，官方路径只有前台服务等合规手段。
  >
  > 追问：被杀后恢复——ActivityRecord 在 system_server，返回时重启进程（链接 Activity 异常恢复链）。

* 面试官视角

  > **考察点**：ART 混合编译、文件格式链路、Zygote COW、oom_adj——是否把 runtime 与启动/内存串起来。
  >
  > **加分**：Profile-Guided 与 Baseline Profile、vdex 避免重复 verify、单线程 fork 安全性。
  >
  > **减分**：只知道"ART 比 Dalvik 快"；文件格式混淆；不知道 Zygote 预加载。

---

* Android ClassLoader 类型与双亲委派机制

  > **类型**：
  > - **BootClassLoader**：加载 framework 类（java.*、android.*）。
  > - **PathClassLoader**：加载已安装 APK（dex 路径固定）。
  > - **DexClassLoader**：可加载任意路径的 dex/jar/apk/so，曾是插件化/热修复基础（现需注意 API 26+ 对可写 dex 限制与隐藏 API）。
  > - **InMemoryDexClassLoader**（API 26+）：内存 dex，免落盘。
  >
  > **双亲委派**：loadClass 先问 parent，parent 没有才 findClass 自己加载——保证核心类唯一（防止恶意替换 java.lang.String），代价是"子加载器无法覆盖父已加载类"。
  >
  > **Android 特点**：PathClassLoader/DexClassLoader 的 parent 是 BootClassLoader；应用类加载器通常自成一体。热修复与插件化正需要"打破或利用"委派（见下题）。

* 热修复主流方案对比（类加载与底层原理）

  > 1. **底层替换（Native hook，如 Sophix 早期思想/Tinker 部分）**：改 ArtMethod 结构体指针/native 替换方法入口——即时生效、无需重启；对 ART 内部结构敏感、版本兼容难。
  > 2. **类加载方案（QZone/AndFix 演进/Tinker）**：把修复 dex 插在 DexClassLoader 前面，利用**双亲委派的 findClass 顺序**：若父加载器未加载过该类，优先从补丁 dex 加载——**需重启生效**（类已加载无法替换）。
  > 3. **Instant Run/资源/So 修复**：资源 id 表重定向、so 替换（System.loadLibrary 路径劫持）。
  >
  > 面试要点：类加载方案的"插 dex 前面"本质是控制 findClass 搜索顺序；底层替换涉及 ArtMethod 布局随 Android 版本变化，是兼容性黑洞。合规提醒：热修复涉及动态下发代码，应用市场政策收紧（需走合规通道）。

* 面试官视角

  > **考察点**：ClassLoader 与双亲委派是否真懂，热修复能否讲清"生效时机与原理差异"。
  >
  > **加分**：DexClassLoader 与 PathClassLoader 差异、类加载方案需重启的原因、ArtMethod 兼容性。
  >
  > **减分**：把热修复当黑盒"打补丁就行"；说不清为什么即时生效难；不知道双亲委派。
