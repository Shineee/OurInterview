# Java 函数式接口

函数式接口（Functional Interface）是 Lambda 的目标类型：**只有一个抽象方法**的接口。Java 8 在 `java.util.function` 包提供了大量通用函数式接口，避免人人自定义单方法接口。

## 核心五个（必须张口就来）

| 接口 | 方法签名 | 语义 | 典型场景 |
| ---- | ---- | ---- | ---- |
| `Predicate<T>` | `boolean test(T t)` | 断言：进 T 出布尔 | 过滤 `filter` |
| `Function<T,R>` | `R apply(T t)` | 转换：进 T 出 R | `map` |
| `Consumer<T>` | `void accept(T t)` | 消费：进 T 无返回 | `forEach` |
| `Supplier<T>` | `T get()` | 供给：无参出 T | 懒创建、`Supplier` 型字段 |
| `UnaryOperator<T>` | `T apply(T t)` | 一元运算：进 T 出 T | 替换/规范化 |

另有二元变体：`BiFunction<T,U,R>`、`BiPredicate<T,U>`、`BiConsumer<T,U>`、`BinaryOperator<T>`（如 `BinaryOperator.maxBy(comparator)`）。

```java
Predicate<String> notEmpty = s -> s != null && !s.isEmpty();
Function<String, Integer> len = String::length;
Consumer<String> println = System.out::println;
Supplier<List<String>> newAL = ArrayList::new;
UnaryOperator<String> trim = String::trim;

// 组合（接口上的 default 方法是面试加分点）
Predicate<String> p = notEmpty.and(s -> s.length() > 3);   // 与
Predicate<String> q = notEmpty.or(s -> "skip".equals(s));  // 或
Predicate<String> r = notEmpty.negate();                   // 非

Function<Integer, Integer> add1 = x -> x + 1;
Function<Integer, Integer> doubleIt = x -> x * 2;
Function<Integer, Integer> pipeline = add1.andThen(doubleIt);  // 先 add1 再 double：(1+1)*2
Function<Integer, Integer> pipeline2 = add1.compose(doubleIt); // 先 double 再 add1：(1*2)+1
```

## default 方法速查（函数式接口的"组合能力"）

- `Predicate`：`and`、`or`、`negate`、`isEqual`。
- `Function`：`andThen`（后置）、`compose`（前置）——注意方向，是高频易错点。
- `Consumer`：`andThen`（先执行本消费再执行下一个）。
- `UnaryOperator`/`BinaryOperator`：`identity()`；`BinaryOperator.minBy/maxBy(Comparator)`。
- 这些 default 方法让"函数管道"可以在不引入第三方库的情况下拼装。

## 原始类型特化（避免装箱）

泛型会装箱（`int` → `Integer`），热点路径有性能与内存代价。`java.util.function` 提供原始类型特化：

| 泛型 | 原始特化 |
| ---- | ---- |
| `Predicate<Integer>` | `IntPredicate`（`test(int)`，返回 boolean） |
| `Function<Integer,R>` | `IntFunction<R>`、`ToIntFunction<T>`、`LongFunction`、`DoubleFunction` |
| `Consumer<Integer>` | `IntConsumer`、`LongConsumer`、`DoubleConsumer` |
| `Supplier<Integer>` | `IntSupplier`、`LongSupplier`、`DoubleSupplier` |
| `UnaryOperator<Integer>` | `IntUnaryOperator`、`LongUnaryOperator`、`DoubleUnaryOperator` |
| `BiFunction<Integer,Integer,R>` | `IntBinaryOperator` 等 |

注意 `IntFunction<R>`（进 int 出 R）与 `ToIntFunction<T>`（进 T 出 int）方向不同——签名读题时别急。

**面试追问：`ToIntFunction<T>` 为什么要存在？** → `Function<T, Integer>` 返回的是包装类（装箱分配）；`ToIntFunction<T>` 返回原始 `int`，在 Stream 的 `mapToInt` 路径上避免装箱，数值密集计算时减少 GC 压力。

## 常用派生接口（Stream 配套）

- `UnaryOperator<T>` 就是 `Function<T,T>` 的别名式重命名（`Function<T,T>` 也可，但语义更明确）。
- `Comparator` 本身就是函数式接口（`compare` 单抽象方法），可用 Lambda：`Comparator.comparing(String::length)`。
- `Runnable`、`Callable<V>` 也是函数式接口（后者常配合 CompletableFuture/线程池）。

## 自定义函数式接口

```java
@FunctionalInterface
public interface Parser<T> {
    T parse(String raw);                     // 唯一抽象方法

    default T parseOrDefault(String raw, T def) {
        try {
            return parse(raw);
        } catch (Exception e) {
            return def;
        }
    }                                        // default 不破坏函数式判定

    static Parser<Integer> ofInt() {
        return Integer::parseInt;            // 静态方法同样不计入抽象方法
    }
}

Parser<Integer> p = Integer::parseInt;       // 方法引用即实现
```

## 设计判断：何时自定义 vs 用标准接口

- **优先用标准接口**：`Predicate`/`Function`/`Consumer` 组合性好、可被 Stream/Optional 复用。
- **自定义接口**当语义需要被"点名"（提高可读性、避免 `Function<String, X>` 满天飞）、或需要该接口特有的默认方法时。
- **避免**为一个方法的接口引入继承层次——函数式接口的价值就在扁平、可组合。

## 【面试追问链】

1. **接口里有 default 方法还算函数式接口吗？** → 算。判定标准是"抽象方法（不含 Object 继承来的）恰好一个"；`Comparator`、`Predicate` 全靠 default 方法获得组合能力。
2. **一个接口继承两个函数式接口会怎样？** → 若抽象方法签名兼容则合并为一个，仍可能是函数式接口；若冲突（同名不同返回类型）则不再是——继承结构里"恰好一个抽象方法"是硬约束。
3. **`Function.andThen` 与 `compose` 谁先执行？** → `f.andThen(g)` 先 f 后 g；`f.compose(g)` 先 g 后 f。口诀：andThen 是"然后"，compose 是"由……组成（先内后外）"。写反了是常见 Bug。
4. **为什么 Lambda 能同时适配多个函数式接口？** → 目标类型由赋值上下文决定，同一段 `x -> x + 1` 可推断为 `Function<Integer,Integer>` 也可为 `UnaryOperator<Integer>`——多目标重载时可能引发推断歧义，需显式类型。
5. **`Supplier` 和工厂方法有什么区别？** → 语义上 Supplier 表示"延迟/可重复的供给"，常用于 `Optional.orElseGet(Supplier)`、缓存懒加载；普通工厂方法只是一次性调用。用 Supplier 是为了**推迟求值时机**（见 Optional 的 orElse vs orElseGet）。
6. **函数式接口能用 `@Override` 吗？** → 实现 Lambda 时无所谓（没有方法体）；用匿名类实现时可以写 `@Override`。接口内 default/static 方法不是抽象方法，与 `@Override` 无关。
