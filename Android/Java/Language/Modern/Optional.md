# Java Optional

`Optional<T>` 是 Java 8 引入的**显式空值容器**：把"值可能存在也可能不存在"编码进类型系统，迫使调用方处理缺失分支，替代满天飞的 `null` 与 `NullPointerException`。核心价值是**语义**，不是魔法——用错会比直接 null 更糟。

## 创建

```java
Optional<String> a = Optional.of("hello");        // 值必须非 null，否则 NPE
Optional<String> b = Optional.empty();            // 明确的空
Optional<String> c = Optional.ofNullable(maybeNull());  // 可 null，null 则 empty

// 不要写：Optional.of(null)  —— 直接 NPE
// 不要把 Optional 当"可能 null 的字段"的容器到处传 null 进 of
```

## 正确消费方式

```java
Optional<String> opt = Optional.ofNullable(findName());

// 1. 有值才执行 / 无值走默认（二选一，必有结果）
String v1 = opt.orElse("default");           // 无条件求值默认值（见下文 orElseGet）
String v2 = opt.orElseGet(() -> compute());  // 仅在空时求值

// 2. 有值才执行副作用
opt.ifPresent(v -> System.out.println(v));
opt.ifPresentOrElse(v -> use(v), () -> fallback());

// 3. 变换（空则仍为空——Optional 自动短路）
Optional<Integer> len = opt.map(String::length);
Optional<String> upper = opt.filter(s -> s.length() > 3).map(String::toUpperCase);

// 4. 必须有值，否则抛异常（明确约定"此处不允许空"）
String v3 = opt.orElseThrow(() -> new IllegalStateException("name required"));

// 5. 判断与取值（最不推荐的手写风格）
if (opt.isPresent()) { String v = opt.get(); }   // 能用上面的 API 就别手写
```

**链式是 Optional 的正确形态**：`map/filter/orElse/orElseThrow` 让缺失处理沿着管道自然传播，避免中途 `get()` 前忘记 `isPresent()`。

## 反模式（面试重点）

### 反模式 1：`orElse` vs `orElseGet`

```java
Optional<User> opt = loadUser(id);              // loadUser 可能有成本

// orElse：无论 opt 是否为空，参数都会被**立即求值**
User u1 = opt.orElse(new User(defaultName()));  // defaultName() 每次都执行

// orElseGet：仅在空时调用 Supplier
User u2 = opt.orElseGet(() -> new User(defaultName()));  // 有值时零成本

// 错误示范：orElse 里放查询
opt.orElse(db.findDefaultUser());   // 即使 opt 有值也会打一次 DB！
```

规则：

- 默认值是**常量/廉价表达** → `orElse`（可读性好，差异可忽略）。
- 默认值需要**计算、IO、DB、new** → 必须 `orElseGet(Supplier)`，否则白白执行。
- 两者语义区别不是"性能小优化"，而是**求值时机**：`orElse` 参数在调用点求值（eager），`orElseGet` 惰性。

### 反模式 2：用 Optional 作字段 / 参数类型

```java
// 反例
class User {
    Optional<String> name;        // 不要：Optional 不是为字段设计的
}

// 反例：方法参数收 Optional
void process(Optional<String> input) { ... }
```

原因：

- 字段/参数的"可空"本就是 Java 常态（`null` 或注解 `@Nullable`）；把 Optional 放进字段会污染序列化（JSON 库对 Optional 支持不一）、equals/hashCode、框架注入（Spring 字段注入对 Optional 的支持语义模糊）。
- **Optional 的设计场景是"方法返回值"**：告诉调用方"可能没有，必须处理"。
- 约定：参数用普通类型（必要时 `@Nullable`），返回值用 `Optional` 表达缺失。

### 反模式 3：`get()` 不判断

```java
opt.get();   // 空时抛 NoSuchElementException —— 和裸 null 差不多，还多了层误导
```

`get()` 只有在已经用 `isPresent()`/`ifPresent` 确认、或类型上保证非空时才可使用；一般直接换 `orElseThrow`。

### 反模式 4：Optional 当集合用

```java
// 反例：Optional<List<String>> 表达"可能没有元素"
// 正例：空 List 就是空 List；Optional 只表达"元素本身可能缺失"
```

`Optional` 不是 `List` 的替代品：集合为空是正常状态，`isEmpty` 即可；Optional 表达的是**单个值**的缺失。

### 反模式 5：在 Optional 上做 `isPresent + get` 双遍历逻辑

```java
if (opt.isPresent() && opt.get().length() > 3) { ... }   // 冗余
// 改为
opt.filter(s -> s.length() > 3).ifPresent(s -> ...);
```

## Optional 的实现要点（一句话版）

- 本质是 `Optional`（有值）与 `Optional.Empty`（单例）两个实现；`map`/`filter` 空值短路返回 `Empty`，不执行函数。
- `Optional.map` 内部如果函数返回 null 会转为 `empty`（不会嵌套出 `Optional<null>`）——与 `Stream.map` 行为不同（Stream 允许 null 元素）。
- `Optional` 本身不可变、无状态，可安全复用（但没有理由跨线程共享一个"结果容器"）。

## 何时用 Optional 的决策

```
方法返回值可能缺失，且调用方必须显式处理 → Optional<T>
参数 → 普通类型（null 约定 + 注解）
集合/数组 → 空集合语义，不是 Optional
字段 → 普通类型；序列化/框架边界尤其避免
需要默认值 → orElse（廉价）/ orElseGet（昂贵）/ orElseThrow（必须有）
```

## 【面试追问链】

1. **`orElse` 和 `orElseGet` 到底差在哪？** → 求值时机：`orElse` 的实参在调用点无条件计算；`orElseGet` 仅在空时通过 Supplier 计算。默认值昂贵（DB/IO/复杂构造）时用错会每次都白算——这是语义差异，不只是微优化。
2. **为什么 Optional 不设计成字段类型？** → 它为"返回值缺失语义"而生；进字段会与序列化格式、框架绑定、equals/hashCode、JPA 持久化等基础设施冲突，且字段空值本来就有成熟约定（null + 注解）。返回值 Optional 让调用方在编译期看到"要处理缺失"。
3. **`Optional<Optional<T>>` 说明什么设计问题？** → 说明嵌套缺失语义——几乎总是糟糕设计的信号；`flatMap` 正是用来压平嵌套的（`opt.map(x -> innerOpt)` 出现嵌套时改用 `flatMap`）。
4. **`map` 里返回 null 会怎样？** → `Optional.map` 把函数返回的 null 转成 `empty`，链式继续安全传播——所以 `opt.map(key -> map.get(key))` 不会 NPE，但注意它与 `opt.map(map::get)` 的语义都可能是"键不存在→empty"。
5. **方法什么时候不该返回 Optional？** → 性能极端敏感的热点（Optional 创建有少量开销——虽然很便宜）、需要表达"多种失败原因"的场景（应抛异常或返回 Result 类型）、以及同方法里"偶尔 null 合法"的 API（保持 null 一致性）。
6. **`ofNullable(x).orElseThrow()` 和直接 `Objects.requireNonNull(x)` 有什么区别？** → 语义相近（都要求非空）；`requireThrows` 在参数阶段更常见，Optional 链适合"值经过多次变换后"再断言。关键是**约定一致**：要么处处 Optional，要么核心路径用异常/断言，不要两种风格混杂。
