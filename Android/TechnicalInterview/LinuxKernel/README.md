# Linux Kernel 面试题

* Binder 为什么设计成"一次拷贝"？与 Socket/共享内存对比

  > **传统 IPC（如管道/Socket）**：发送方拷贝到内核缓冲，接收方再从内核拷到用户空间——两次拷贝。
  >
  > **Binder**：接收方进程在打开 /dev/binder 时 **mmap** 了一块内核缓冲；发送方写入时，Binder 驱动直接把数据拷贝到这块**接收方可见的 mmap 区域**——只有一次从发送方用户空间到接收方映射区的拷贝。
  >
  > **对比**：共享内存零拷贝但无同步/无权限模型，难用；Socket 双拷贝+灵活。Binder 在"易用（同步调用语义）+性能（1 拷贝）+安全（UID 鉴权）"间取得平衡，这是 Android 选它做核心 IPC 的原因。
  >
  > 面试加分：提到 mmap 大小约 1MB-8KB/进程（链接 Activity Bundle 1MB 限制）。

* Binder 架构中的四个角色：Client / Server / ServiceManager / 驱动

  > - **Client**：发起调用（如 App 调 AMS）。
  > - **Server**：实现服务（system_server 里的 AMS/PMS）。
  > - **ServiceManager**：名片簿（handle 0），Client 通过它按名字查 Server 的 Binder 引用（addService/getService）。
  > - **Binder 驱动**：内核模块，负责跨进程引用转发、引用计数、死亡通知（DeathRecipient）、数据搬运。
  >
  > 调用链：Client → 查 ServiceManager → 拿 Server 的 BpBinder → transact → 驱动路由到 Server 的 BBinder → onTransact → 返回。
  >
  > 面试追问：为什么需要 ServiceManager？——跨进程不能直接传对象指针，需要一个"全局已知"的入口（handle 0）做名字→引用的映射，类似 DNS。

* Binder 线程池与 oneway/同步调用的差异

  > Server 端默认 Binder 线程池最多 **16 线程**（可配），处理并发 transact；实现需线程安全。
  >
  > **同步 transact**：调用方线程阻塞等待 reply（主线程慎用，易死锁/ANR）。
  > **oneway**：异步，不等 reply，不阻塞调用方；用于回调/通知（如 Input 事件分发部分路径）。oneway 无返回值、无异常回传（异常经 oneway 时特殊处理受限）。
  >
  > 死锁场景：主线程同步调 Server，Server 又同步回调主线程——经典 Binder 递归死锁；排查看 systrace binder 状态。

* Binder 通信中的数据大小限制与传输大对象的策略

  > 单次事务受接收方 mmap 缓冲限制（约 1MB 共享），且是进程级共享池——并发多事务挤压时更小（链接 TransactionTooLargeException）。
  >
  > 大数据策略：文件描述符（ParcelFileDescriptor）传 fd 让对方 mmap/read；ashmem 共享内存传 region；分片传输；或干脆走文件/ContentProvider 流式接口。
  >
  > 面试串联：Bitmap 跨进程不直接塞 byte[]，系统剪贴板大文本用共享内存——都是这个限制的工程应对。

* 死亡通知（DeathRecipient）与 Binder 连接管理

  > Client 可对 BpBinder linkToDeath 注册 DeathRecipient，Server 进程死亡时驱动回调 binderDied——用于重连 Service、清理缓存引用。
  >
  > 为什么需要：跨进程引用不会因为对端死了自动让 Client 知道，若不监听，后续 transact 返回 DeadObjectException 但业务层可能已状态错乱。AIDL 客户端模板常用 it.linkToDeath。
  >
  > ServiceConnection.onServiceDisconnected 也是这层机制的应用（系统帮你在 bindService 场景封装）。

* 面试官视角

  > **考察点**：Binder 是否到驱动与 mmap 层级，能否解释 Android 为何选它、限制与工程应对。
  >
  > **加分**：1MB 共享池、16 线程池、oneway 死锁、DeathRecipient、ServiceManager 的 DNS 类比。
  >
  > **减分**：只知道"Binder 是 Android 的 IPC"；说不清一次拷贝在哪；把共享内存当 Binder。

---

* Ashmem（匿名共享内存）是什么、解决什么问题、如何使用

  > **Ashmem**：Android 内核驱动（/dev/ashmem），提供**可跨进程共享、可 pin/unpin 回收**的匿名内存。比 mmap 匿名共享更灵活：拥有者可 unpin 释放物理页（内容丢），系统内存紧张时也可回收 unpin 区域——适合"大数据跨进程共享但不需要长期占物理内存"。
  >
  > 使用：MemoryFile（老 API）/ ParcelFileDescriptor + Ashmem；Binder 传 fd，对方 mmap 使用。
  >
  > 典型场景：系统剪贴板大文本、跨进程传大 Bitmap、相机/媒体 buffer 共享（现代多被图形栈取代）。面试要点：Ashmem 是"带回收策略的共享内存"，不是普通 mmap。

* ION / DMA-BUF 在 Android 图形栈中的角色

  > **ION**（旧）：Android 统一的** dma-buf 分配器**管理器，给相机、GPU、编解码器分配可跨设备/进程共享的缓冲（heap：system/cma/secure 等）。不同硬件 IP（GPU/Camera/Video）通过 dma-buf fd 共享同一块物理内存，避免拷贝。
  >
  > **DMA-BUF**：Linux 内核标准的缓冲共享框架；Android 新内核（5.10+ 趋势）用 **dma-heap** 替代 ION（ION 上游被拒，官方迁移到 dma-heap）。
  >
  > 面试串联：SurfaceFlinger 合成、Camera HAL 出图、Codec 解码——都经 GraphicBuffer/dma-buf 在进程间传递 fd，零/少拷贝。能讲到"图形栈用 dma-buf 共享 buffer 而不是 Binder 拷贝字节"即内核深度加分。

* SELinux 在 Android 中的作用（简述）

  > **MAC（强制访问控制）**：传统 Linux DAC（owner/group 权限）之外，Android 引入 SELinux 对**进程域（domain）**与**对象类型**做策略强制——即使 root 也受策略限制（防提权后为所欲为）。
  >
  > 应用层体现：每个 app 域受限（不能随意访问其它 app 数据、设备节点）；未定义策略的操作直接 denied（avc: denied 日志）。
  >
  > 对开发的影响：访问特殊设备/文件/属性被拒时看 **audit log + sepolicy**；系统定制/ROM 移植常要写 sepolicy 规则。面试简述即可：知道 DAC+MAC、域隔离、防提权目的。

* Binder + Ashmem + SELinux 串起来的 Android 安全模型

  > - Binder：天然带调用方 UID/PID，Server 可鉴权（"谁在调我"）。
  > - Ashmem/DMA-BUF：大数据共享走 fd，不经 Binder 拷贝字节，但 fd 传递本身仍受 Binder 权限控制。
  > - SELinux：进程域能力边界，即使拿到 fd/ Binder 也要策略允许。
  >
  > 三层叠加回答了"Android 如何在开放 IPC 下保证安全"——面试官若问"Android 安全机制"，从 IPC 鉴权讲到 MAC 比只讲签名/权限高级。

* 面试官视角

  > **考察点**：共享内存与图形栈 buffer 流转、SELinux 定位，是否把内核机制与上层框架串起来。
  >
  > **加分**：Ashmem pin/unpin、dma-heap 替代 ION、Binder UID 鉴权、audit log 排查 denied。
  >
  > **减分**：Ashmem/共享内存概念混淆；认为 SELinux 只是"root 权限管理"；不知道图形 buffer 走 fd。
