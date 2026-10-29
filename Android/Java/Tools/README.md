# Java 常用工具与诊断

排查线上问题、准备 JVM/性能类面试时，工具链是必须熟悉的。本页按"一句话用途 + 典型场景"整理 JDK 自带工具与主流图形化诊断工具。

## JDK 命令行工具

以下工具位于 JDK 的 `bin` 目录下（JDK 9+ 部分工具迁移到 `jcmd`/`jhsdb` 等，但老工具大多仍可用）。

| 工具 | 一句话用途 | 典型场景 |
| ---- | ---- | ---- |
| `jps` | 列出本机正在运行的 Java 进程及其 PID | 拿到目标进程 PID，作为其他工具的输入 |
| `jstat` | 输出 GC、类加载、编译等运行时统计信息 | `jstat -gc <pid> 1000` 每秒观察堆占用与 GC 频率，判断是否频繁 Full GC |
| `jmap` | 生成堆转储（heap dump）或查看堆内存分布 | `jmap -histo:live <pid>` 看哪类对象占内存多；`jmap -dump:format=b,file=heap.hprof <pid>` 导出堆供 MAT 分析 |
| `jstack` | 打印线程栈快照 | 排查 CPU 飙高（找到忙线程栈）、死锁（输出 `Found one Java-level deadlock`） |
| `jinfo` | 查看/修改 JVM 运行时参数与系统属性 | 确认线上进程实际生效的 `-Xmx`、GC 参数，或动态开启 GC 日志 |

补充：

- `jcmd <pid> VM.flags` / `jcmd <pid> GC.heap_dump`：JDK 7u4+ 推荐的统一诊断入口，功能覆盖 jstat/jmap/jinfo 的一部分。
- `jar` / `javap`：`javap -c` 反编译字节码，验证语法糖（如 `finally`、字符串拼接）的真实实现。
- `javadoc`、`jshell`（JDK 9+）：文档生成与交互式验证代码片段。

**面试追问：如何定位一次线上 OOM？**

1. 启动参数加 `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/path` 让 JVM 自动导出堆。
2. 用 `jmap -histo:live` 或 MAT 的 Dominator Tree 找占用最大的对象链路。
3. 结合代码检查大集合未清理、缓存无上限、流未关闭、ThreadLocal 未 remove 等常见泄漏点。

**面试追问：jstack 看到 `WAITING (parking)` 和 `BLOCKED` 有什么区别？**

`BLOCKED` 表示线程在等待获取监视器锁（synchronized）；`WAITING (parking)` 多为 `LockSupport.park`、线程池 `take`、`Condition.await` 等无锁等待。前者指向锁竞争，后者多指向正常的任务队列等待或条件等待。

## 图形化与商业工具

- **VisualVM**：JDK 自带（JDK 9 起需单独安装），可监控 CPU/内存/线程、采样、生成堆 dump、做性能剖析（Profiler）。轻量场景的首选。
- **JProfiler**：商业工具，功能更细——方法级耗时、调用树、内存分配热点、线程状态分析，支持远程附加。试用期 10 天。
- **YourKit**：与 JProfiler 齐名的商业 Profiler，UI 与快照对比功能成熟。
- **MAT（Eclipse Memory Analyzer）**：专门分析 hprof 堆转储，擅长泄漏报告（Leak Suspects）与支配树（Dominator Tree）。
- **Arthas**：阿里开源的线上诊断神器，`dashboard`、`watch`/`trace` 看方法入参返回值与调用耗时、`jad` 反编译线上类、`redefine` 热更新，无需重启进程。
- **async-profiler**：低开销采样 Profiler，可产出火焰图，定位 CPU 热点与锁等待分布。

## Android Profiler（简述）

Android Studio 自带的 Profiler 提供：

- **CPU Profiler**：方法追踪（Trace）与采样，看主线程卡顿/热点方法。
- **Memory Profiler**：堆快照、分配追踪，排查内存泄漏（配合 LeakCanary 更佳）。
- **Network Profiler**：请求时序、响应体查看（OkHttp/Retrofit 流量）。
- **Energy Profiler**：耗电归因。

命令行侧还有 `adb shell dumpsys meminfo <pkg>`、`systrace`/`Perfetto`（系统级帧与调度分析）可配合使用。

## 复习建议

工具类面试常考的组合是：**jps 拿 PID → jstat 看 GC → jmap/jstack 定位对象与线程 → MAT/Arthas 分析**。把这条链路亲手跑一遍，胜过背诵命令参数。
