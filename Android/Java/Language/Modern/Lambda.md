# Java Lambda 表达式

Lambda 是 Java 8 引入的**匿名函数**：把"行为"当作参数传递，让代码更简洁地表达"做什么"而不是"怎么做"。它是函数式接口的语法糖。

## 语法

```
(参数列表) -> 表达式          // 体只有一条表达式（自动成为返回值）
(参数列表) -> { 语句块 }      // 多条语句需要花括号与显式 return
参数类型 -> ...               // 参数类型通常可推断，可省略
x -> x * x                   // 单参数可省略圆括号
```

```java
// 匿名类写法
Runnable r1 = new Runnable() {
    @Override
    public void run() {
        System.out.println("hello");
    }
};

// Lambda 等价写法
Runnable r2 = () -> System.out.println("hello");

// 带参函数式接口
Function<String, Integer> f = s -> s.length();
Predicate<String> notEmpty = s -> !s.isEmpty();
```

要点：

- **参数类型可省略**，由目标类型（函数式接口）推断；需要显式声明时全部参数都要写类型（不能只写一部分——这是与匿名类不同的语法限制）。
- **单表达式**可省略 `return`（隐式返回）；**语句块**里必须显式 `return`。
- Lambda **不是关键字重载**，不会引入作用域隔离的新层级——见下文变量捕获。

## 函数式接口（Functional Interface）

- 只有**一个抽象方法**的接口才能用 Lambda 实现（可用 `@FunctionalInterface` 标注让编译器检查）。
- 可以有多个**默认方法/静态方法**——不影响函数式判定（如 `Comparator` 有大量 default 方法但仍是函数式接口）。
- 标准函数式接口见 [FunctionalInterface](FunctionalInterface.md)：`Predicate`、`Function`、`Consumer`、`Supplier`、`UnaryOperator` 等。
- 任意只有单抽象方法的自定义接口都可以是 Lambda 的目标。

**面试追问：`@FunctionalInterface` 是必须的吗？** → 不是。它只让编译器校验"确实只有一个抽象方法"，不写也能编译通过；写上是良好习惯，能防止后续改接口时不小心破坏 Lambda 兼容。

## 变量捕获

Lambda 可以访问**外部局部变量**，但有严格限制：

```java
int factor = 3;                          // effectively final
Function<Integer, Integer> f = x -> x * factor;   // OK：捕获 factor

// factor = 4;                           // 编译错误：一旦被 Lambda 捕获必须 effectively final

Runnable r = () -> {
    int local = 1;                       // Lambda 内部可以声明自己的局部变量
    System.out.println(local);
};
```

规则：

1. 捕获的局部变量必须是 **final 或 effectively final**（赋值后不再改变）——否则多线程/延迟执行下值不确定。
2. Lambda 内部**可以**声明同名局部变量，但内部变量遮蔽（shadow）外部变量，不是同一个。
3. 捕获的是**变量的值（对对象而言是引用）**，不是"活的绑定"——之后修改对象内容可见，重新给变量赋值不可行（编译都过不了）。
4. **字段（实例变量/静态变量）不受此限制**——Lambda 可随意读写 `this.x`，因为它捕获的是 `this` 引用。

## Lambda 与匿名内部类的区别

| 维度 | Lambda | 匿名内部类 |
| ---- | ---- | --- |
| 语法 | 函数式语法糖，极简 | 必须写完整类体 |
| 目标类型 | 只能是**函数式接口**（单抽象方法） | 任意接口/抽象类/具体类 |
| this 语义 | 指向**外围**的 this | 指向匿名类**自身**实例 |
| 局部变量 | 捕获必须 effectively final | 同样要求 final/effectively final（Java 8 前是显式 final） |
| 独立类文件 | 编译为 `lambda$main$0` 方法（无独立 .class） | 生成独立的 `Outer$1.class` |
| 字段/初始化块 | 不允许 | 允许（它是真正的类） |
| 实现机制 | invokedynamic 动态调用点 | 编译期生成匿名类字节码 |

**面试追问：Lambda 里能有实例字段和静态方法吗？** → 不能。Lambda 不是类，只是实现函数式接口的"行为"；需要状态就用外部字段或用 `AtomicReference` 等捕获对象。

**面试追问：Lambda 里的 `this` 指谁？** → 指**外层**对象（Lambda 没有自己的 this）。匿名类里 `this` 是匿名类实例——这是两者最容易踩的语义差异。

## 实现原理（一句话到位）

Lambda **不生成匿名内部类字节码**，而是在调用点编译成 `invokedynamic`（JDK 8+ `LambdaMetafactory`）：首次执行时由 Bootstrap 方法生成一个实现了目标接口的**调用站点（CallSite）**，直接把 Lambda 体链接为一个私有静态/实例方法的句柄；之后走的是一次方法句柄调用，**创建开销远低于反复 new 匿名类**（无额外对象分配时甚至可内联）。这就是"Lambda 比匿名类高效"的底层原因，也是 Java 8 选择 invokedynamic 而不是直接生成类文件的动机——把"生成实现类"的决定权留给运行时，便于后续优化与字符串动态调用（`invokedynamic` 也是 Groovy/JRuby 等 JVM 语言共享的基础设施）。

## 方法引用（Method Reference）

Lambda 的简写：`Class::method`、`instance::method`、`String::length`。

```java
list.forEach(System.out::println);        // 等价于 s -> System.out.println(s)
Function<String, String> upper = String::toUpperCase;   // 实例方法，接收者作首参
Supplier<LocalDate> today = LocalDate::now;
```

四种形式：静态 `A::m`、实例方法 `A::m`（未绑定）、绑定实例 `obj::m`、构造器 `A::new`。

## 使用场景与注意

- 集合遍历/变换：配合 [Stream](Stream.md) 使用。
- 策略模式、回调（`Comparator`、`Runnable`、自定义 Listener）。
- **序列化陷阱**：Lambda 不是匿名类，不能直接序列化；需要传递可序列化行为时用 `Serializable` 标记的函数式接口或显式匿名类。
- 不要滥用单行 `->` 堆砌业务逻辑：超过一两行的 Lambda 建议抽方法（方法引用），可读性优先。

## 【面试追问链】

1. **为什么 Lambda 要限制 effectively final？** → Lambda 可能在创建后的任意时刻执行（异步、延迟），若捕获可变变量，执行时的值与创建时的预期可能不一致且存在数据竞争；编译期禁止从根上消除这类 Bug。
2. **Lambda 比匿名类高效在哪？** → 匿名类每次执行 `new` 都分配对象、加载类文件；Lambda 通过 invokedynamic 链接方法句柄，可复用、可内联，分配成本趋近于零。
3. **只有单抽象方法就一定是函数式接口吗？** → 还要接口本身（非 Object 方法）只有这一个抽象方法；继承来的 `equals`/`toString` 不算。`Comparator` 有几十个 default 方法但只有 `compare` 一个抽象方法，所以成立。
4. **Lambda 能替换所有匿名类吗？** → 不能：需要状态字段、需要独立 `this`、目标是抽象类而非接口、需要序列化时，仍用匿名类（或具体类）。
5. **`this` 在 Lambda 与匿名类中为何不同？** → Lambda 没有类外壳，编译后方法句柄绑定到外围实例，`this` 自然是外层的；匿名类是真实类，`this` 指向自身。写回调时若在内部调用外围方法，两者行为差异会直接暴露。
6. **字符串拼接为什么也用 invokedynamic？** → JDK 9 起 `StringConcatFactory` 用 `makeConcatWithConstants` 的 invokedynamic 替代 `StringBuilder` 预编译，运行时可按情况选择最优拼接策略——与 Lambda 共享同一套动态调用基础设施。
