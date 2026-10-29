# Android 内核层笔试题

* 什么是 Binder？用一句话说明原理，并讲清"一次拷贝"是怎么实现的。

  > **一句话**：Binder 是 Android 基于内核驱动 + mmap 实现的跨进程 RPC 机制：数据从发送方用户空间拷贝一次到内核缓冲区，而接收方将同一物理页 mmap 到自己的用户空间，从而省掉第二次拷贝，同时驱动层记录调用方真实 UID/PID 完成可信身份传递。
  >
  > **一次拷贝的完整路径（机制级）**
  > 1) 通信双方各自通过 /dev/binder 的 mmap 把一块物理内存（通常约 1MB-8KB）映射进自己的用户空间；
  > 2) Client 发起事务：copy_from_user 把数据从 Client 用户空间拷贝到驱动持有的内核缓冲区——这是唯一一次拷贝；
  > 3) 因为接收进程 mmap 的是同一批物理页，数据"已经"出现在接收方用户空间，无需 copy_to_user；
  > 4) 驱动按 BINDER_TYPE_* 解析 flat_binder_object，处理 Binder 引用的跨进程传递（把实体/引用翻译成对端可用的 handle）。
  >
  > 对比：管道/Socket 需要 写入内核 + 读出内核 两次拷贝；共享内存零拷贝但无身份与同步语义，需自行加锁与鉴权。
  >
  > 【追问】为什么是约 1MB？——binder 驱动给每个进程 mmap 的空间有限，一次事务总大小受此约束，超限抛 TransactionTooLargeException；因此大负载必须转 ashmem/fd。

* 为什么 Android 选择 Binder 而不用传统 IPC？（要求从性能、安全、架构三角度列举）

  > **性能**：一次拷贝，优于管道/消息队列/Socket 的两次拷贝，接近共享内存；且传输的是结构化对象引用而非裸字节流。
  >
  > **安全**：驱动在内核态记录调用方真实 UID/PID，接收方无需也无法伪造调用者身份——传统 IPC 中 socket/管道的身份认证要靠传递凭证，容易出错。配合 permission 机制，系统服务可按 uid 鉴权。
  >
  > **架构**：C/S 模型清晰；AIDL 编译期生成 Proxy/Stub 样板代码，调用像本地方法；支持 death recipient（linkToDeath）感知对端进程死亡并清理引用；ServiceManager 以 handle 0 提供名字服务，形成完整的服务注册-查询-调用闭环。
  >
  > **代价（体现权衡）**：驱动与 AIDL 复杂度高；同步调用会阻塞调用方线程，滥用于主线程即 ANR；对高频小包场景不如共享内存高效——所以媒体等大数据路径走 Ashmem。

* Ashmem 是什么？与 Binder 是什么关系？

  > Anonymous Shared Memory：基于内核 ashmem 驱动的共享内存，允许不同进程映射同一块物理内存，实现接近零拷贝的大块数据共享。
  >
  > 与 Binder 的配合：Binder 传大数据受 1MB 事务上限约束，实际做法是把 ashmem 的 fd 通过 Binder 传递，数据本体走共享内存——Binder 传"钥匙"，Ashmem 传"货物"。典型：Bitmap 跨进程传递（Parcel.writeBlob / MemoryFile）、媒体编解码缓冲。
  >
  > 特性：支持 pin/unpin，内存紧张时内核可回收未 pin 的页并标记丢弃（读到 0 填充），比 mmpa 匿名共享更"有礼貌"。
  >
  > 【追问】Android 11+ 中 ashmem 使用被收紧，新代码推荐 MemoryFile/SharedMemory API——体现对版本演进的了解。

* 用户空间与内核空间是什么概念？一次系统调用与一次 Binder 调用各跨越什么边界？

  > CPU 特权级划分：内核空间拥有全部硬件权限，运行操作系统；用户空间受限，应用进程运行其中，不能直接操作硬件。
  >
  > 系统调用：用户态 -> 内核态（syscall 指令），返回时回到用户态，涉及上下文切换与页表切换开销。
  >
  > Binder 调用链跨越的边界：Client（用户态）-> ioctl 进入内核态（Binder 驱动）-> 可能唤醒 Server 进程（另一个用户态进程的 Binder 线程被唤醒）-> Server 在其用户态处理后 ioctl 返回。一次完整 RPC 经历两次用户/内核切换 + 一次进程调度，但仍只有一次数据拷贝——区分"拷贝次数"与"切换次数"是加分点。

* 简述 Binder 通信的四个角色，以及一次完整调用的时序。

  > Client / Server / Binder 驱动（内核）/ ServiceManager（守护进程，以 handle 0 注册，提供类似 DNS 的 name->handle 查询）。
  >
  > 时序：1) Server 向 SM 注册 Binder 实体；2) Client 向 SM 查询得到目标 handle（实为驱动翻译出的引用）；3) Client 通过 ioctl 发起 transaction，驱动校验、拷贝数据、找到目标进程的 todo 队列；4) 目标进程 Binder 线程池（默认最多 16 线程，由 BINDER_SET_MAX_THREADS 控制）取事务，经 Stub.onTransact 分发到业务代码；5) reply 沿原路返回。
  >
  > 加分点：主线程不能阻塞在同步 Binder 上过久（ANR）；oneway（异步）事务不等待 reply，走异步队列，仍受目标进程缓冲区配额限制。
  >
  > 【追问】为什么 Client 看起来在"调用本地方法"？——AIDL 生成的 Proxy 把参数打包进 Parcel 走 transact，Stub 端 onTransact 解包调用真实实现，异常也通过 reply 传回，形成 RPC 透明性。

* 为什么说 Binder 传输数据有大小限制？工程上如何处理超限负载？

  > 每个进程 mmap 的 binder 缓冲约 1MB-8KB，且同一时刻所有进行中的事务共享该缓冲；超限即 TransactionTooLargeException（或静默失败为 TransactionTooLarge），且崩溃点常在接收方反序列化时，排查困难。
  >
  > 处理策略：1) 大对象改传 ashmem/SharedMemory 的 fd 或文件路径；2) 分页/流式接口（ContentProvider 的 openFile/query CursorWindow 即为此设计）；3) 传引用不传本体（跨进程用句柄/ID 二次查询）；4) 压缩；5) 架构上避免把大对象塞进 Bundle/Intent（包括嵌套 Parcelable）。
  >
  > 考察点：是否知道 CursorWindow 底层就是共享内存，ContentProvider 查询不把整表拷进目标进程。

* 【方案权衡】跨进程传一张 20MB 的大图，方案 A：分块走 AIDL 多次传输；方案 B：写临时文件传路径；方案 C：Ashmem/SharedMemory。如何选？

  > A（分块 AIDL）：实现直观但每次事务都占用 binder 缓冲并有同步开销，16 次以上事务时延迟叠加，且任一块失败重传复杂——不推荐主用。
  >
  > B（临时文件）：简单可靠、天然支持超大数据；但有磁盘 IO 与权限/清理问题，端到端延迟高，多用户场景有安全风险（文件权限、路径穿越）——适合离线/一次性场景。
  >
  > C（Ashmem）：一次 fd 传递 + 双方 mmap，接近零拷贝，延迟最低，是系统内跨进程传大内存的默认选择；代价是 fd 生命周期管理与驱动可用性（新版 API SharedMemory）。
  >
  > 结论：进程间热路径选 C；持久化或超大离线数据选 B；避免把 A 当作默认。考察的是"延迟 / 复杂度 / 生命周期"三维权衡，而非背 API。
