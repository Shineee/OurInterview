# Kotlin 协程与 Flow（面试主战场）

> 定位：把协程从"会 launch"推到**编译产物、取消与异常矩阵、冷热流与背压语义**。与 `Android/TechnicalInterview/.../Asynchronous` 基础题互补——这里默认你已读过 CPS/结构化并发入门。

## 目录

- [第1页](1.md)：协程本质（状态机/CPS，非线程魔法）；CoroutineContext 组成与合并；launch / async / runBlocking / withContext 语义边界；结构化并发（父子 Job、取消传播、Supervisor）
- [第2页](2.md)：取消协作（挂起点、CE 不可吞、finally/NonCancellable）；异常传播矩阵（launch/async/withContext/Flow）；CoroutineExceptionHandler 边界；Dispatcher 选型与泄漏
- [第3页](3.md)：Flow 冷流与 collect 惰性、flowOn；冷转热（SharedFlow/StateFlow、replay、WhileSubscribed）；背压与 buffer/溢出；flatMapConcat/Latest/Merge；异常；与 LiveData/回调选型
- [第4页](4.md)：共享可变状态陷阱（withContext 不串行化、单线程交错）；Mutex / Actor / Channel；StateFlow.update；测试：TestDispatcher、虚拟时间、advanceUntilIdle

## 使用建议

- 面试口述顺序建议：先第1页建立"状态机 + 作用域"框架，再用第2页矩阵应对追问，第3–4页覆盖 Flow 与并发原语开放题。
- 每页末追问链可单独作为二三轮深挖提纲。
