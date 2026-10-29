# Kotlin（Android 一等公民）

> 定位：把 Kotlin 从「会写」推到「在 Android 工程里讲清编译产物、协程语义与架构用法」。**不按独立 Kotlin/后端语言栈叙述**——协程、Flow、状态持有、与 View/Compose 互操作、KMP 都落在 Android 应用与架构决策上。

每题按「问题 → 机制级答案 → 为什么这样设计 → 面试官视角 → 追问链」组织，与 [Android Senior](../Senior)、[Java Senior](../Java/Senior) 同一深度标准。

## 在 Android 面试中的位置

- 协程 / Flow：异步、生命周期、状态上卷的主语言设施（对照旧 Handler/AsyncTask 叙事）。
- 语言机制：空安全在 Parcelable/JSON/Java 互操作边界；inline/value class 在列表热路径上的成本意识。
- 架构与工程：viewModelScope、StateFlow、KMP 渐进共享——资深面与方案设计面的高频词。

## 目录导引

### [语言深挖（Language）](Language)

- 第1页：空安全（平台类型）、data class、object/伴生、sealed 与 API 演进
- 第2页：inline/noinline/crossinline、reified、value class、委托、扩展、contracts
- 第3页：高阶函数与 Sequence、型变、与 Java 互操作十大坑（Android 模块边界高频）

### [协程与 Flow（Coroutines）](Coroutines) —— 面试主战场

- 第1页：状态机/CPS、CoroutineContext、构建器边界、结构化并发
- 第2页：取消与异常传播矩阵、Dispatcher 选型与泄漏
- 第3页：Flow 冷热、stateIn/shareIn、背压、与 LiveData 选型
- 第4页：共享可变状态、Mutex/Actor、测试（虚拟时间）

### [Android 实战（Android）](Android)

- 第1页：viewModelScope/lifecycleScope/repeatOnLifecycle；StateFlow/SharedFlow vs LiveData
- 第2页：View/Compose 互操作、Parcelable 空安全、热路径性能、编译产物与包体积

### [工程收束（Senior）](Senior)

- 第1页：KMP 答法、何时不用协程、与 JVM 虚拟线程对照
- 第2页：模块 API 空安全策略、二进制兼容、编译速度——tech lead 叙事

## 使用建议

- 先 Language 建地基，再 Coroutines 打主战场，最后 Android + Senior 收束到架构与团队工程叙事。
- 与 [Java](../Java) 对照读：协程不替代线程/内存模型知识；互操作坑两边各写一半才完整。
