# BroadcastReceiver 面试题

* 静态注册与动态注册的区别、各自使用场景与限制

  > **静态注册（Manifest）**：安装后常驻，应用未启动也能接收（系统会拉起进程）。适合系统级事件（开机 BOOT_COMPLETED、应用更新、语言变化）。限制：Android 8+ 大部分隐式广播不再静态接收（见下题）；需注意滥用被 Play 政策限制。
  >
  > **动态注册（registerReceiver）**：跟随组件生命周期，必须 unregister 否则泄漏。适合应用内事件（网络变化、自定义动作）。Context.registerReceiver 与 LocalBroadcastManager（已废弃）不同。
  >
  > 生命周期：动态注册在 onReceive 时进程可能已在前台；静态注册触发冷启动拉起进程——"广播拉活"曾是保活手段，现已收紧。

* 有序广播与无序广播的区别，以及 abortBroadcast 的意义

  > **无序广播（sendBroadcast）**：异步发给所有匹配接收者，无先后顺序保证，不可拦截。
  >
  > **有序广播（sendOrderedBroadcast）**：按 priority 从高到低依次传递，接收者可：
  > - setResult / getResultExtras 传递数据给下一接收者。
  > - **abortBroadcast**（需权限）拦截，后续低优先级接收者收不到。
  >
  > 系统广播如 SMS_RECEIVED 是有序的（高优先级短信应用可先拦截）；自定义有序广播用于"链式处理+可取消"场景。注意并发与性能：有序广播同步链路长会拖慢发送方。

* LocalBroadcastManager 为什么被废弃？替代方案是什么

  > LocalBroadcastManager 曾用于应用内广播：不跨进程、高效、安全（不暴露给外部应用）。
  >
  > **废弃原因**：官方认为它与全局广播 API 混淆，且应用内通信有更好方案；它内部用 Handler 实现，粘性广播（已废弃 API）等语义与全局广播不一致，易误用。
  >
  > **替代**：
  > - 应用内单向数据：LiveData / StateFlow（生命周期感知）。
  > - 事件总线：SharedFlow（注意粘性/replay 语义）或 EventBus（需注销）。
  > - 跨进程本地：显式广播 + 权限，或 ContentProvider/AIDL。
  >
  > 面试要点：LocalBroadcast 本质是"进程内 Handler 分发"，不是 Binder IPC，理解这点就知道为何它比全局广播快、也为何语义有差异。

* Android 8.0+ 对隐式广播的限制及绕过方案

  > **限制**：为省电与减少唤醒，应用不能在 Manifest 静态注册接收**大部分隐式广播**（没有明确指定包名的广播）。例外：自家应用发的显式广播、系统白名单广播（BOOT_COMPLETED 等少数）、有 ACTION 触发的豁免清单。
  >
  > **影响**：以前靠静态监听网络变化、电量变化拉活的做法全部失效。
  >
  > **合规替代**：
  > - 动态注册（应用活着时）。
  > - JobScheduler/WorkManager 的 NetworkType 约束（系统在满足约束时拉起你的 Job，而非广播）。
  > - 前台服务 + 动态监听。
  >
  > 面试考察：是否理解这是"系统级功耗治理"而非 bug，能否给出 WorkManager 约束的现代方案。

* 显式广播与隐式广播的定义与匹配规则

  > **显式**：setComponent(ComponentName) 或 setPackage 指定目标，直接点对点。
  >
  > **隐式**：靠 IntentFilter 匹配（action + category + data）。匹配要求 action 匹配（注册了多个 action 任一即可）、category 必须全部匹配（Intent 的 category 是 filter 的子集语义——filter 必须声明 Intent 里的所有 category）、data 按 scheme/host/path 匹配。
  >
  > 安全：隐式广播若 export 且无权限，任何应用可发——重要广播需加 permission（senderPermission）或用显式+包名。

* 面试官视角

  > **考察点**：注册方式生命周期、有序广播控制流、8+ 功耗治理对开发模式的影响。
  >
  > **加分**：LocalBroadcast 废弃的深层原因（非 Binder）、WorkManager NetworkType 替代、IntentFilter 三要素匹配细节。
  >
  > **减分**：还在推荐静态监听网络变化保活；认为 LocalBroadcast 是"轻量 Binder"；说不清 category 匹配规则。

---

* onReceive 在哪个线程执行？广播超时 ANR 怎么判定

  > onReceive 默认在**主线程**执行（动态注册与静态注册都一样，除非 goAsync + 自建线程）。超过主线程处理超时会触发广播 ANR：**前台广播 10s / 后台广播 60s**（BroadcastQueue 超时监控）。
  >
  > 因此 onReceive 里不能做网络/磁盘等耗时操作；要异步处理用 goAsync() 或把任务转给 Service/WorkManager。
  >
  > 追问："静态注册的广播拉起冷启动，10s 从何时算？"——从分发给你的进程开始算；冷启动 Application.onCreate 慢会挤占这个预算，这也是"广播触发冷启动 ANR"的常见原因（Application 初始化过重）。

* goAsync 的语义与正确用法

  > onReceive 返回后系统认为"处理完成"，若需在后台线程继续（如写 DB、上报），用 `PendingResult r = goAsync()` 拿到异步句柄，在子线程处理完调 `r.finish()`。
  >
  > 注意：goAsync **不延长无限时间**，仍在广播超时预算内（前台 10s）；子线程必须 finish 否则泄漏。重任务应 startService/WorkManager，而不是把广播当任务队列。

* 广播接收与进程拉起的关系（冷启动路径）

  > 静态注册 + 隐式/显式广播命中 → AMS 发现目标进程不存在 → Zygote fork → Application.onCreate → 创建 Receiver 实例 → onReceive。
  >
  > 这曾是"保活链"核心（互相拉活），现被 8+ 隐式限制、国产 ROM 后台管理砍断。面试可讲"广播拉活"历史与现状，体现对系统演进的理解。
  >
  > 注意：系统对广播拉起的进程也计入 oom_adj，空进程优先回收；不要依赖它做关键服务。

* 广播安全：权限、保护性 API 与常见漏洞

  > 1. **发送方加权限**：sendBroadcast(intent, receiverPermission)——只有声明该权限的接收者才能收。
  > 2. **接收方声明权限**：receiver 标签 android:permission 限制谁可发。
  > 3. **显式 + 包名**：自定义广播尽量 setPackage 限制范围，防外部应用伪造。
  > 4. **exported 属性**：Android 12+ 组件必须显式声明 exported=true/false；广播 receiver 默认对系统广播 exported，自定义勿随意 true。
  >
  > 常见漏洞：导出的 receiver 未鉴权，接受外部"命令类"action（如清数据、改配置）→ 被恶意应用触发。审查点：所有 exported receiver 的 action 是否敏感、是否有签名级权限保护。

* 网络状态监听的演进：BroadcastReceiver → ConnectivityManager.NetworkCallback

  > 旧方案：动态注册 CONNECTIVITY_ACTION，收到后判断网络。缺点：广播异步延迟、Android 7+ 对 CONNECTIVITY_ACTION 静态限制。
  >
  > 新方案：`ConnectivityManager.registerDefaultNetworkCallback` / registerNetworkCallback，直接回调 onAvailable/onLost/onCapabilitiesChanged，更实时且支持多网络（Wi-Fi+蜂窝并存时的默认网络切换）。
  >
  > 面试可对比：回调式 API 比广播更精准，官方全面推荐；再叠加 WorkManager 的 NetworkType 约束做"有网才执行"的任务调度。

* 面试官视角

  > **考察点**：线程与 ANR 预算、goAsync 的边界、安全审查意识、网络监听的现代 API。
  >
  > **加分**：冷启动拉活历史与政策、exported 显式声明、NetworkCallback 能力回调（不仅 onAvailable）。
  >
  > **减分**：在 onReceive 做网络请求；以为 goAsync 可以无限后台跑；还在用 CONNECTIVITY_ACTION 静态注册。
