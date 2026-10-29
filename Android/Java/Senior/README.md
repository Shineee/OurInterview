# Java 高阶（Android 语境下的专家深水区）

> 定位：资深/专家 Android 面试里从「业务代码」挖到「并发、内存、分布式客户端问题」时的深度答案。**不是后端 Java 专场**——每节都尽量挂回 Android：线程池与主线程/后台任务治理、GC 与 ART 对照、缓存一致性与客户端重试幂等。

每题按「问题 → 机制级答案 → 为什么这样设计 → 面试官视角 → 追问链」组织，与 [Android Senior](../../Senior)、[Kotlin](../../Kotlin) 同一标准。

## 目录

### [并发（Concurrency）](Concurrency)

- 第1页：AQS 源码级问答（状态、队列、独占/共享、Condition）；线程池设计与调优——对照 Android 线程池治理、后台任务与背压
- 第2页：Disruptor 与无锁队列概念；并发 bug 排查（死锁、可见性）——对照 jstack / 主线程阻塞 / ANR 归因思路

### [JVM 工程（JVMEngineering）](JVMEngineering)

- 第1页：GC 选型与停顿叙事；内存泄漏排查（堆转储、GC Root）——与 ART GC、LeakCanary 方法论同构
- 第2页：JIT 与逃逸分析；元空间/ClassLoader 泄漏；虚拟线程 vs 线程池 vs Kotlin 协程的取舍话术

### [客户端分布式常见题（DistributedJava）](DistributedJava)

- 第1页：分布式 ID、缓存一致性、幂等与去重、消息可靠投递——移动端对应：本地缓存失效、请求重试、推送通道 ACK

## 使用建议

- Android 候选人必吃透 Concurrency + JVMEngineering；DistributedJava 至少掌握缓存一致性与幂等两节。
- 回答时主动把 JVM 结论**翻译到 ART/Android 约束**（不可插拔收集器、堆档位、主线程模型），是专家区分度。
