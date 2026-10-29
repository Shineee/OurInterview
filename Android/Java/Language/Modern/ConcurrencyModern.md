# Java 现代并发（CompletableFuture / StampedLock / 虚拟线程）

Java 8 之后并发工具的三条主线：**异步编排**（CompletableFuture）、**读写性能优化**（StampedLock）、**线程模型革命**（Project Loom 虚拟线程）。本页按"面试怎么答"组织，机制与权衡优先。

## CompletableFuture：异步组合

`Future` 只能 `get()` 阻塞等待，无法表达"两个都完成再做、任一完成先做、失败降级"。`CompletableFuture`（Java 8）实现了 `Future` + `CompletionStage`，把异步任务组织成**依赖图**。

### 创建

```java
CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> loadFromDb());
CompletableFuture<String> cf2 = CompletableFuture.supplyAsync(() -> callApi(),
        executor);                       // 显式指定线程池；默认用 ForkJoinPool.commonPool()

CompletableFuture<String> done = CompletableFuture.completedFuture("ok");
CompletableFuture<String> failed = CompletableFuture.failedFuture(new IOException("x")); // Java 9+
```

**注意**：`supplyAsync` 默认走公共 ForkJoinPool——**阻塞 IO 任务会占满公共池**、拖垮同池所有并行流；IO 密集务必传自定义 `ExecutorService`。

### 组合（面试主战场）

```java
CompletableFuture<User> userF = asyncLoadUser(id);
CompletableFuture<Orders> ordersF = asyncLoadOrders(id);

// 1. 两者都完成再组合（并行发出，join 时汇合）
CompletableFuture<Profile> profile = userF.thenCombine(ordersF,
        (user, orders) -> new Profile(user, orders));

// 2. 依赖链：上一步结果喂给下一步（同步/异步两版）
CompletableFuture<String> result = userF
        .thenApply(User::getName)                 // 同步变换（当前线程/完成线程执行）
        .thenCompose(name -> asyncQuery(name));   // 返回 Future 时必须 flatten

// 3. 消费副作用（无返回值）
userF.thenAccept(u -> log.info("loaded {}", u));
userF.thenRun(() -> cache.warm());

// 4. 任一完成 / 全部完成
CompletableFuture.anyOf(cf1, cf2, cf3).thenApply(Object::toString);
CompletableFuture.allOf(cf1, cf2).thenRun(() -> allReady());

// 5. 超时与降级（Java 9+）
CompletableFuture<String> withTimeout = cf
        .completeOnTimeout("fallback", 2, TimeUnit.SECONDS)
        .exceptionally(ex -> "degraded");

// 6. 异常处理
cf.exceptionally(ex -> defaultValue);
cf.handle((value, ex) -> ex == null ? value : fallback());
```

**`thenApply` vs `thenCompose`**（高频追问）：

- `thenApply`：`Future<T> → Future<U>`（同步函数，类似 `Optional.map`）。
- `thenCompose`：`Future<T> → Future<Future<U>>` 压平（类似 `Optional.flatMap`）。当下一步本身返回 `CompletableFuture` 时必须用 `thenCompose`，否则得到嵌套的 `CompletableFuture<CompletableFuture<U>>`。

**执行线程**：`thenApply/thenAccept` 等在**触发完成的那个线程**上执行（已完成则调用线程直接执行）；要强制异步用 `*Async` 变体（`thenApplyAsync`，无 executor 版用公共池，可传自定义池）。

### 与线程池、JUC 的关系

- 异步任务最终都落到 `Executor`；生产上应自建有界队列 + 命名线程的池，避免公共池饥饿。
- `CompletableFuture` 可与 `Stream`/`Optional` 类比理解：组合子（map/compose/filter 对应）把回调地狱压成声明式管道。
- 反面：不要在 `CompletableFuture` 链里做长阻塞 IO 而不指定独立池；不要用 `join()` 在持有锁时等待（死锁风险）。

## StampedLock（概念级）

`ReentrantReadWriteLock` 的读写锁在高并发读下有 **CAS 读计数 + 队列** 的开销，且写者可能饥饿。Java 8 的 `StampedLock` 提供**乐观读**：

```java
StampedLock lock = new StampedLock();

// 写锁
long stamp = lock.writeLock();
try {
    x = 1.0;
} finally {
    lock.unlockWrite(stamp);
}

// 乐观读：不阻塞写者，读完用 stamp 校验期间是否被写过
long stamp = lock.tryOptimisticRead();
double v = x;                          // 无锁读共享变量
if (!lock.validate(stamp)) {           // 读期间发生过写 → 升级为悲观读锁重读
    stamp = lock.readLock();
    try { v = x; } finally { lock.unlockRead(stamp); }
}
```

要点（面试答法）：

- **乐观读**：读时不加锁，返回一个版本戳（stamp）；读完 `validate` 检查版本，失败则回退为普通读锁——"先信后验"，把"读多写少且读极快"场景的开销压到接近无锁。
- **不可重入**：`StampedLock` 不是 `ReentrantLock` 的替代，不支持重入，用错容易自死锁；也不支持 Condition 的 `await` 语义（有 `readLockInterruptibly` 等变体但功能子集）。
- 适用：读极多、写少、临界区短的统计类共享变量（类似 `LongAdder` 的思路但保护任意结构）。
- 对比：`ReadWriteLock` 语义简单可重入；`StampedLock` 更快但更难用；无锁结构仍以 `Atomic*` / `ConcurrentHashMap` 为主。

## 虚拟线程（Project Loom，Java 21 正式）

### 面试怎么答（三句话结构）

1. **是什么**：JDK 21 正式 GA 的轻量级线程——由 JVM 调度到少量平台线程（carrier）上，**阻塞时不占用载体线程**（会 unmount），创建成本极低（可百万级）。
2. **解决什么**：传统平台线程 = OS 线程，1:1 映射、创建贵、上下文切换贵、连接数受线程数限制——为每请求一线程的 IO 密集服务（HTTP、JDBC、RPC）付出巨大内存与调度代价。
3. **怎么用**：`Thread.ofVirtual().start(() -> ...)` 或 `Executors.newVirtualThreadPerTaskExecutor()`；代码**几乎不变**（仍是阻塞式写法），不需要响应式/回调改造。

```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 100_000; i++) {
        executor.submit(() -> {
            Thread.sleep(Duration.ofSeconds(1));   // 阻塞！不占用载体线程
            return i;
        });
    }
}   // 关闭时等待所有任务完成
```

### 关键机制与边界（体现"学通"）

- **调度**：ForkJoinPool 的 work-stealing 调度器把虚拟线程 mount/unmount 到载体线程；阻塞的 `synchronized`/IO/JDBC 等已在 JDK 内逐步改为**可挂载点**（JEP 虚拟线程的 pinning 问题在 JDK 21+ 显著改善）。
- **不要池化虚拟线程**：创建太便宜，"每任务一个"是设计初衷；池化平台线程的旧习惯反而限制并发。
- **synchronized 不再是"性能禁忌"**：虚拟线程下锁竞争的代价与普通方法差异变小，但**长临界区**仍会占用载体线程、降低吞吐——锁的粒度与不变性依然重要。
- **适用**：大量并发阻塞 IO（网关、爬虫、JDBC 长查询）。**不适用**：CPU 密集（载体线程数 = CPU 核，跑计算任务虚拟线程不会提速，反而增加调度开销）。
- **与 Kotlin 协程对比（加分）**：协程是**语言/运行时**层面的挂起点（需要编译器配合、`suspend` 函数），虚拟线程是 **JVM** 层面的线程，对既有阻塞代码透明；两者都把"轻量并发"从线程数限制中解放出来。
- **ThreadLocal 注意**：百万虚拟线程 × ThreadLocal = 内存放大；优先用 `ScopedValue`（Java 21 预览/后续版本推进）传递上下文。

### 相关 JVM 并发演进一句话

`synchronized` 锁升级（偏向→轻量→重量）与 `java.util.concurrent` 的 AQS/CLH 队列是并发基础；`VarHandle` 替代 `sun.misc.Unsafe` 提供内存语义；GC 侧的分代假设与 ZGC/Shenandoah 的低停顿演进则从**系统层面**支撑高并发服务——机制之间是相互配合的（面试可把"并发 + 内存模型 + GC"串成一条线）。

## 【面试追问链】

1. **`thenApply` 和 `thenCompose` 什么时候必须用后者？** → 下一步返回 `Future` 时；用 `thenApply` 会得到嵌套 `CompletableFuture<CompletableFuture<T>>`，语义从"变换"变成"包装"，`get()` 拿到的还是 Future。
2. **CompletableFuture 默认线程池有什么坑？** → `ForkJoinPool.commonPool` 线程数 = CPU-1 且与并行流共享；IO 阻塞占满后全局饥饿。生产必须显式传有界队列的 `ExecutorService`。
3. **`exceptionally` 和 `handle` 差在哪？** → `exceptionally` 只处理异常路径（成功则透传）；`handle(value, ex)` 成功失败都执行，适合"无论成败都要记日志/清理"。
4. **StampedLock 的乐观读适合什么、不适合什么？** → 适合读远多于写、读临界区极短、可接受"validate 失败重读"的场景；不适合需要条件等待、重入、长读操作（validate 失败率高反而更慢）。
5. **虚拟线程会让 `synchronized` 变快吗？** → 不是变快，而是"不再那么贵"：阻塞时挂载释放载体，传统"锁内不能阻塞"的教条松动；但长临界区仍占住载体、吞吐下降，锁纪律不能丢。
6. **虚拟线程适合 CPU 密集吗？** → 不适合。载体线程数受 CPU 核限制，计算任务不会因虚拟线程变多而并行度提高；它解决的是"海量等待"而不是"海量计算"。
7. **什么时候不该用 CompletableFuture 而该用响应式/手动 Future？** → 简单依赖链与超时降级用 CF 足够；需要背压、流式处理、复杂取消传播时再考虑 Reactor/RxJava——不要为了响应式而响应式。
