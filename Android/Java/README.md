# Java（Android 工程师向）

> **定位**：Android 应用跑在 JVM/ART 语义之上，Java 是读源码、排内存/并发问题、理解字节码与类加载的语言地基。本目录**不按独立 Java 后端栈叙述**，一切以「Android 工程师需要讲清楚什么」为取舍：语言与集合 → 并发与内存 → 与 Android 运行时（ART/dex/ClassLoader）对齐的工程能力。

每题深度标准与 [Android Senior](../Senior) 一致：机制级答案 → 为什么这样设计 → 面试官视角 → 追问链（高阶树）。

# 在 Android 面试中的用法

- **应用层追问**：HashMap 与线程安全、synchronized/volatile、对象分配与 GC——面试官常从业务代码挖到语言与运行时。
- **排查叙事**：内存泄漏、卡顿、崩溃的根因常落在 Java 对象图、线程模型与 GC 行为上。
- **与 Kotlin 对照**：协程建模在 JVM 线程与内存模型之上；互操作坑（空平台类型、默认参数）见 [Kotlin](../Kotlin)。

# 深度复习

### [Language](Language) —— 语法、面向对象、集合、算法、Java 8+（Lambda/Stream）

### [JVM](JVM) —— 类文件、字节码、加载链接初始化（对照 ART 时说语义差异）

### [Senior](Senior) —— 并发 AQS/线程池、GC 与泄漏排查、缓存一致与幂等（Android 侧同构问题）

### [Frameworks](Frameworks) —— 常用框架概念（面试偶发）

### [Tools](Tools) —— 诊断工具链（jstack 等，对照 Android Profiler）

# 发展简史（一句话）

1995 年诞生 → 2004 Java 5 里程碑 → 2014 Java 8（Lambda/Stream，Android 语言现代化对应）→ Oracle 时代与 OpenJDK 多发行版。细节不必背年表，面试重在语言版本与 Android API/语言特性对应关系。
