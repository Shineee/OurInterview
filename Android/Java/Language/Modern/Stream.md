# Java Stream

Stream（`java.util.stream`）是 Java 8 引入的**函数式数据处理管道**：对集合做"声明式"的链式操作（过滤、映射、聚合），而不是手写循环。核心心智模型：**流水线（惰性求值） + 操作融合 + 终端才执行**。

## 创建 Stream

```java
// 1. 集合
Stream<String> s1 = List.of("a", "b", "c").stream();
Stream<String> s2 = Stream.of("a", "b", "c");

// 2. 数组
Stream<Integer> s3 = Arrays.stream(new int[]{1, 2, 3}).mapToObj(i -> i);

// 3. 无限流 / 区间（配合 limit 终止）
Stream<Integer> naturals = Stream.iterate(0, n -> n + 1);      // 0,1,2,...
Stream<Double> randoms = Stream.generate(Math::random);
IntStream range = IntStream.range(0, 10);                      // [0,10)
IntStream closed = IntStream.rangeClosed(1, 100);              // [1,100]

// 4. 字符串
IntStream chars = "hello".chars();

// 5. 文件行
try (Stream<String> lines = Files.lines(Path.of("data.txt"))) {
    // ...
}

// 6. 空流与转换
Stream<String> empty = Stream.empty();
IntStream boxed = IntStream.of(1, 2, 3);                       // 原始特化
Stream<Integer> toBox = boxed.boxed();
```

注意：

- **Stream 不能重复使用**：一个 Stream 只能消费一次，二次调用抛 `IllegalStateException`。
- 集合本身**不受影响**——Stream 只是数据的"视图 + 操作描述"。
- `List.of(...)` 不可变；`Arrays.asList` 返回定长 List（见 ArrayList 页）。

## 中间操作与终结操作

| 类型 | 特点 | 例子 |
| ---- | ---- | ---- |
| **中间操作** | **惰性**，返回新 Stream，调用时不处理任何数据 | `filter`、`map`、`flatMap`、`distinct`、`sorted`、`limit`、`skip`、`peek` |
| **终结操作** | 触发实际计算，返回具体结果或 void，**Stream 由此关闭** | `forEach`、`collect`、`reduce`、`count`、`min/max`、`findFirst/findAny`、`anyMatch/allMatch`、`sum/average`（数值流） |

**惰性求值（lazy evaluation）机制**：

- 只写 `stream.filter(...).map(...)` 不会有任何元素被访问——每个中间操作只是往**流水线（Pipeline）** 上挂一个节点（sink）。
- 调用终结操作时，从数据源拉取元素，**依次穿过整条管道**；因此元素是**逐个处理**的，不会先生成整个中间列表。
- 操作可以**融合**：`filter` 后紧接 `map` 对每个元素一次完成，没有中间集合——这是 Stream 优于"先 filter 成 List 再 map 成 List"的根本原因（少分配、可预测的内存）。

```java
List<String> result = list.stream()
        .filter(s -> s.length() > 2)      // 中间：挂载，不执行
        .map(String::toUpperCase)         // 中间：挂载，不执行
        .collect(Collectors.toList());    // 终结：开始拉取、过滤、转换、收集
```

**短路（short-circuit）**：部分终结操作找到答案即**停止拉取**，后续元素不再处理——对无限流也安全：

```java
list.stream().filter(s -> s.length() > 3).findFirst();   // 找到第一个就返回
Stream.iterate(1, n -> n + 1).limit(100).count();        // 中间 limit 也是短路型：够了就停
list.stream().anyMatch(s -> s.startsWith("x"));         // 有一个 true 就停
```

**面试追问：惰性求值 + 短路的典型组合？** → `findFirst`/`findAny`/`anyMatch`/`noneMatch`/`allMatch` + `limit`。对无限流必须用短路终结（`findFirst`）或 `limit`，否则不返回；而非短路的 `count`/`collect` 在无限流上永远不结束。

## map / filter / reduce / collect

```java
List<String> names = List.of("alice", "bob", "charlie");

// filter：过滤
List<String> longNames = names.stream()
        .filter(n -> n.length() > 3)
        .collect(Collectors.toList());            // [alice, charlie]

// map：一一映射
List<Integer> lengths = names.stream()
        .map(String::length)
        .collect(Collectors.toList());            // [5, 3, 7]

// flatMap：一对多展平（每元素出一个流，再合并）
List<String> words = List.of("hello world", "hi");
List<String> tokens = words.stream()
        .flatMap(s -> Arrays.stream(s.split(" ")))
        .collect(Collectors.toList());            // [hello, world, hi]

// reduce：归约（三种形态）
int sum = IntStream.rangeClosed(1, 100).reduce(0, (a, b) -> a + b);   // 5050
Optional<String> joined = names.stream().reduce((a, b) -> a + "," + b);
OptionalInt maxLen = names.stream().mapToInt(String::length).max();

// collect：可变归约（收集到集合/分组/拼接）
Map<Integer, List<String>> byLen = names.stream()
        .collect(Collectors.groupingBy(String::length));
String csv = names.stream().collect(Collectors.joining(", "));

// 数值流特化（避免装箱）
double avg = names.stream().mapToInt(String::length).average().orElse(0);
```

**reduce vs collect**：`reduce` 是无状态归约（组合函数必须结合律友好、通常不可变）；`collect` 用 `Collector`（supplier/accumulator/combiner）支持可变容器（`ArrayList`、`HashMap`）与并行合并，是集合结果的首选。`Collectors.reducing` 是两者桥梁。

**典型陷阱**：

- `map` 里做**有副作用**的操作（`peek` 打印调试除外）在并行下顺序不定——业务逻辑应放进 `map`/`filter` 的纯函数里。
- `distinct`/`sorted`/`limit` 是**有状态**操作：需要缓冲全部（sorted）或已见元素（distinct），内存与并行合并成本更高。
- `collect(Collectors.toList())` 返回的 List **可变且非线程安全**；要不可变用 `toUnmodifiableList()`（Java 10+）。

## 并行流（parallelStream）

```java
long count = list.parallelStream()
        .filter(this::expensiveCheck)
        .count();
```

**机制**：默认 ForkJoinPool（公共池，线程数 = CPU 核数 - 1），把数据源按**最佳分割点**（spliterator）递归切分，多线程各自处理子任务，再按树形合并（collector 的 combiner）。

**适用条件（全部满足才值得并行）**：

1. **数据量大**（经验上 ≥ 1 万量级，分割与线程调度才有摊销空间）。
2. **操作无状态、无共享可变**——`filter`/`map` 纯函数 OK；`StringBuilder` 累加、共享 `ArrayList.add` 就是数据竞争。
3. **不依赖顺序**：`findFirst` 在并行下语义弱化（找"任意一个匹配的先到者"）；`forEachOrdered` 保序但牺牲并行收益。
4. **不是 IO 密集的每元素操作**（并行流不做阻塞 IO 的调度优化；IO 应用异步/CompletableFuture）。
5. **不会用有状态的 `sorted`/`distinct` 造成频繁合并**——不是不能用，而是合并开销可能吃掉并行收益。

**代价与坑**：

- ForkJoin 公共池被慢任务占满会影响**同池的所有并行流**（全局池饥饿）；可自建 `ForkJoinPool` 并 `pool.submit(...).get()` 隔离。
- 公共池线程是 daemon，`ThreadLocal`（如 SecurityContext、日期格式）在并行流线程上不传递——`SimpleDateFormat` 这类非线程安全对象放进 `map` 在并行下必炸。
- 装箱：并行 + `Integer` 流会放大装箱开销，优先 `mapToInt` 等特化。
- 调试断点困难（多线程交错）——`peek` 不是万能调试器，复杂逻辑单步测方法。

**判断口诀**：**默认串行；并行是优化手段，要拿数据证明**。先正确、再测量、后并行。

## 其他常用操作

```java
// 匹配
boolean any = names.stream().anyMatch(n -> n.startsWith("a"));  // 还有 allMatch/noneMatch

// 查找
Optional<String> first = names.stream().filter(n -> n.length() > 4).findFirst();

// 分组/分区/计数
Map<Boolean, List<String>> part = names.stream()
        .collect(Collectors.partitioningBy(n -> n.length() > 4));
Map<String, Long> freq = words.stream()
        .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()));

// 映射再收集
Map<String, Integer> lenMap = names.stream()
        .collect(Collectors.toMap(Function.identity(), String::length));
```

## Stream vs 集合操作的选型

| 场景 | 选择 |
| ---- | ---- |
| 只读遍历一次、逻辑简单 | for/forEach 足够 |
| 多级过滤变换、分组聚合 | Stream 表达力与可读性更好 |
| 需要下标、多次随机访问 | 集合 API/数组 |
| 超大数据、逐条处理 | 流式 IO + 逐条逻辑，别先 `collect` 进内存 |
| 需要复用结果 | `collect` 成集合后再用（Stream 一次性） |

## 【面试追问链】

1. **Stream 和 Java 8 之前的循环本质区别是什么？** → 循环是"怎么做的"命令式控制流；Stream 描述"要做什么"，由框架控制遍历与融合——惰性 + 无中间集合 + 可并行，是把控制权从程序员交给库。
2. **中间操作为什么必须惰性？** → 惰性才能把多个操作融合成一次遍历（fusion），避免每步物化一个 List；也才能实现短路（`findFirst` 后不再拉取）——惰性是短路的前提。
3. **`peek` 和 `map` 有什么区别？** → `map` 替换流中元素；`peek` 只"看一眼"（通常用于调试/副作用），不改元素。`peek` 不是为调试设计的正式 API，副作用在并行下顺序不定。
4. **为什么 Stream 不能复用？** → 内部流水线状态（已消费标志、缓冲）在终结操作后关闭；复用会语义混乱。需要多条管道就多次 `stream()`。
5. **`collect(toList())` 和 `reduce` 什么时候互换？** → 纯归约（求和、拼接、最值）用 `reduce`；要生成可变集合/分组/统计用 `collect`（有 combiner 支持并行合并）。`reduce` 的组合函数必须满足结合律才能高效并行。
6. **并行流一定更快吗？** → 不一定。小数据（分割/合并开销 > 收益）、顺序敏感、共享状态、IO 等待都会更慢甚至出错；`parallel` 是需要 A/B 验证的优化开关，不是风格装饰。
7. **并行流用的线程池和我自建的 `ExecutorService` 是一套吗？** → 不是。默认用 ForkJoinPool.commonPool（可用系统属性 `java.util.concurrent.ForkJoinPool.common.parallelism` 调整）；CPU 密集可自建池隔离，阻塞型工作不要塞进并行流。
