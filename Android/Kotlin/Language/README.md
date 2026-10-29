# Kotlin 语言深挖（专家向）

## 一、空安全：平台类型与安全调用链

### 问题

Kotlin 的空安全是如何在编译期生效的？从 Java 调用/被 Java 调用时，"平台类型"如何破坏空安全契约？

### 机制级答案

- **类型系统分层**：`T` 非空、`T?` 可空。赋值、参数、返回值位置的空性由编译器检查；可空值取成员必须走 `?.` / `?:` / `!!` / 非空断言或智能转换前提。
- **平台类型（Platform Type）**：从 Java（无空注解）互操作得到的类型是 `T!`——编译器不替你选非空/可空，调用点也不强制检查。这是 Java 互操作破坏空安全的根因：**空契约缺失，不是 Kotlin 变"宽松"了**。
- **安全调用链**：`a?.b?.c` 在任一环为 null 整体短路为 null；`a?.b ?: default` 在 b 整体为 null 时给默认。智能转换（smart cast）依赖"不可变局部变量 + 无自定义 getter"等条件；`var` 属性或有并发写风险时编译器拒绝智能转换，需本地 `val` 暂存。
- **`let` 的语义**：`x?.let { use(it) }` 等价于 null 检查后在非空作用域使用 `it`——本质是安全调用 + 函数式绑定，不产生额外线程/调度；`run`/`also`/`apply`/`takeIf` 等作用域函数在空安全场景常与 `?.` 组合。
- **编译产物概念级**：`!!` 编译为 null 检查后抛 `KotlinNullPointerException`（现版本消息可能为 `NullPointerException`，以所用版本为准）；`?.` 编译为条件分支，不是魔法 AST。安全调用链会被编译成嵌套条件，不引入闭包捕获开销（相比显式 `let` 更轻）。
- **Java 互操作空注解**：Kotlin 识别 `org.jetbrains.annotations.Nullable/NotNull`（`@Nullable`/`@NotNull`）及 JSR-305/`androidx.annotation` 等常见注解（视 Kotlin 版本与注解约定）。在 Java API 上标注后，Kotlin 侧按非空/可空处理；反向暴露 Kotlin API 时，字节码层可用 `@Nullable`/`@NotNull` 或 `@JvmSuppressWildcards` 等配合——**对外 Java API 必须写注解，否则平台类型回流**。

### 【为什么这样设计】

可空性进类型系统而不是运行时 assert，是把一类 NPE 从线上事故前移到编译错误；平台类型是渐进采纳 Java 生态的务实折衷——没有注解就无法声明契约，宁可不撒谎也不假安全。

### 【面试官视角：考察点 / 加分 / 减分】

- 考察点：平台类型来源、`?.` vs `!!`、智能转换限制、Java 注解如何收回平台类型。
- 加分：区分"Kotlin 编译期检查"与"Java 侧仍可能传 null"；提到 `@JvmField`/属性 getter 命名对智能转换的影响；提到自定义模块间 API 用注解 + 显式可空性表达契约。
- 减分：宣称"Kotlin 里不可能 NPE"；把 `!!` 当默认风格；不知道平台类型。

### 【追问链】

1. `var` 局部变量为什么有时仍能智能转换？→ 无并发检测的局部可变，若作用域内无法被其他线程写且无重新赋值，则可转换；跨 lambda 可变捕获会阻断。
2. 平台类型最危险的场景？→ 链式调用 `dao.find(user).name` 且 `find` 来自 Java 返回 `null`——编译通过，运行 NPE。
3. 如何强制 Java 调用方传非空？→ 参数可空性 + 运行时检查（`requireNotNull`）或生成带注解的 API；编译期无法约束 Java。

---

## 二、data class 语义与 copy 陷阱

### 问题

`data class` 自动生成了什么？`copy` 在可变属性/嵌套引用上有哪些隐蔽陷阱？

### 机制级答案

- **自动生成**：`equals`/`hashCode`（按主构造属性）、`toString`、`copy`（默认参数为当前值）、按声明顺序的 `component1..N`（解构 `val (a, b) = x`）。`equals` 基于结构相等（值），非引用相等。
- **不变性建议**：主构造用 `val`——`var` + `copy`/外部 setter 会让 `equals` 与哈希行为不可预期；放入 `HashSet`/`HashMap` 后被"原地修改"会破坏去重与查找。
- **copy 陷阱**：`copy` 是浅拷贝——只复制属性引用。嵌套可变集合（`MutableList`）仍共享同一实例：`b = a.copy(); b.tags.add("x")` 会改到 `a.tags`。深不可变需拷贝集合或改用不可变集合类型。
- **默认参数与 copy**：`copy()` 复制时依赖合成的默认参数机制；若属性过多、copy 频繁，注意对象分配成本（可考虑 builder/专用工厂，但多数业务场景可忽略）。
- **与 Java Bean**：data class 不是 JavaBean；`copy`/`componentN` 对 Java 侧不可见为常规方法。序列化（Parcelable/Moshi/Gson）需按属性路径正确支持——见 Android 树。

### 【为什么这样设计】

data class 把"值对象"的样板方法收敛到语言层，强迫主构造承载语义字段；浅拷贝是 JVM 引用模型下的默认，深拷贝无法在不知集合语义时替用户猜——陷阱本质是"值语义外观 vs 引用语义内核"。

### 【面试官视角：考察点 / 加分 / 减分】

- 考察点：equals/hashCode 字段集合、copy 浅拷贝、解构与 componentN、var 风险。
- 加分：主动提"放进 HashMap 键集后改字段导致丢条目"；区分 `copy` 与克隆接口；提到不可变集合库（`kotlinx.collections.immutable`）的工程取舍。
- 减分：以为 `copy` 是深拷贝；在 data class 里放业务行为却不控制 equals 范围（多余字段导致 equals 变严/变松）。

### 【追问链】

1. `equals` 要不要包含所有主构造字段？→ 应用语义上"相等"的字段；把缓存/日志字段放主构造会误判不等。
2. 子类 data class？→ data class 不能是 sealed/data 的子类限制需记准确：data class 可继承 sealed 父；data class 本身 final（不可被继承），要多态用 sealed 层级。
3. hashCode 冲突怎么办？→ 冲突可接受，equals 兜底；不要自定义劣质 hashCode。

---

## 三、object、伴生对象与 Java 静态语义差异

### 问题

`object` 声明、`companion object` 与 Java `static` 有何本质差异？如何正确与 Java 互操作？

### 机制级答案

- **object 声明**：单例——线程安全的实例在类加载时初始化（JVM 上由静态内部类/ holder 模式保证）；可实现接口、有名字，是对象不是"静态方法命名空间"。
- **伴生对象**：每个宿主类一个"伴生"成员，语法上可像静态那样 `Host.create()` 调用——但它是**伴生对象实例的成员**，不是 JVM static。可实现接口（如工厂 `Factory`），可有扩展；`JvmStatic`/`JvmName` 改变字节码暴露。
- **与 Java 调用**：
  - 默认：Java 需 `Host.Companion.method()`；`object` 单例用 `Host.INSTANCE.method()`。
  - `@JvmStatic`：在伴生对象或 object 方法上生成真正的 JVM static 方法（方便 Java）。
  - `@JvmField`：属性暴露为 public 字段；`const val` 生成 `public static final` 常量（仅基本/String）。
- **陷阱**：伴生对象持有宿主引用不会（伴生不反向强引用宿主实例）；但 object 单例生命周期 = 进程，持有 Context 会泄漏。`companion object` 内代码不随"使用某个静态方法"而延迟以外的方式变化——初始化仍在类初始化路径。

### 【为什么这样设计】

伴生对象是"有类型的一等对象"，比 Java static 更可扩展（接口、扩展、泛型）；`JvmStatic` 等注解是互操作逃生舱——语言语义优先，字节码暴露可选。

### 【面试官视角：考察点 / 加分 / 减分】

- 考察点：Companion ≠ static、`INSTANCE`、`@JvmStatic`/`@JvmField`/`const val`。
- 加分：提到伴生对象可实现接口做工厂；提到单例与 DI 的张力（测试替身）；提到 `@file:JvmName` 影响文件类名。
- 减分：说"companion 就是 static"；Java 侧 API 没设计却硬让调用方写 `Companion`。

### 【追问链】

1. 如何让 Java 像用静态一样用伴生？→ `@JvmStatic` + 必要时 `@JvmField`；库代码默认考虑互操作。
2. object 单例测试？→ 接口抽象 + 可替换实现，或避免在核心逻辑直接依赖 object。
3. 嵌套 object 与 sealed 的关系？→ 各自独立；sealed 子类可用 object 表达单例分支。

---

## 四、sealed 在 when 穷尽与 API 演进

### 问题

`sealed class/interface` 在 `when`、状态建模与库 API 演进中解决什么问题？

### 机制级答案

- **语义**：sealed 层级限制子类必须在编译期已知（同包/同模块，以版本演进规则为准）；子类枚举封闭 → `when` 作为表达式可**穷尽检查**：新增分支不处理则编译失败。
- **状态建模**：`sealed class Result<out T> { data class Ok<T>(...); data class Err(...); data class Loading; }` —— 穷尽 `when` 保证 UI/分发层处理所有状态；对比用 `enum` + 可选数据的松散建模，sealed 可携带每态不同载荷。
- **API 演进**：库作者用 sealed 表达"结果/事件"封闭集合；**破坏性**：新增子类对下游 when 表达式是源码级破坏（需更新分支）——库侧应倾向接口 + 文档约定，或接受 major 升级；应用内模块用 sealed 收益最大。
- **接口 sealed**：可跨文件（编译单元规则内）实现，比 class 更轻；`when` 同样可穷尽。Java 侧看作普通类层级，无 sealed 约束感（字节码/元数据层面）。

### 【为什么这样设计】

封闭层级把"漏处理一种状态"从运行时 default 静默吞掉变成编译错误——穷尽性是 Kotlin 对"状态机可维护性"的语言级投入；代价是演进僵硬，故适用域是应用内与稳定协议，而非频繁演化的远程 API。

### 【面试官视角：考察点 / 加分 / 减分】

- 考察点：when 表达式穷尽、与 enum 差异、库演进的破坏性认知。
- 加分：提到 sealed interface 与多实现；提到"库新增分支 vs 应用新增分支"的不同代价；与 Result 类型/函数式错误处理关联。
- 减分：不知道 when 表达式会强制覆盖；到处 default `_ ->` 抹掉穷尽价值。

### 【追问链】

1. when 语句 vs 表达式？→ 表达式必须穷尽；作语句时编译器提示但语义弱（仍建议处理全）。
2. 与 enum 如何选？→ 无关联数据/固定集合用 enum；每态带不同字段用 sealed。
3. Java 消费 sealed？→ 当普通类型用，instanceof 链；穷尽性只在 Kotlin 侧。

---

## 一、inline 函数：noinline / crossinline

### 问题

`inline` 函数解决了什么？`noinline`、`crossinline` 各自解决什么编译/语义限制？

### 机制级答案

- **inline 本质**：编译期把函数体与**函数类型的实参 lambda**内联到调用点，消除 lambda 对象分配与部分虚调用（概念级；并非所有调用都保证无开销，受编译器决策与大小影响）。对高阶函数（参数含 `() -> Unit` 等），收益最明显：循环内小 lambda 不再每次分配。
- **禁止内联的场景编译器会拒绝或警告**：函数内有本地函数递归限制、公开 API 过大导致调用方 dex 膨胀——`inline` 标记是请求，编译器可拒绝非 lambda 场景的激进内联（lambda 参数的内联是设计保证）。
- **noinline**：对某个 lambda 参数禁止单独内联该参数——当 lambda 需要作为**对象传递**（存字段、传给非内联 API、反射调用）时，保留其函数对象形态。
- **crossinline**：允许内联进**非局部返回受限**的上下文（如嵌套 lambda/局部对象中），代价是禁止非局部 `return`（`return` 只能返回内联函数自身，不能"从外层命名函数返回"）。解决"lambda 内联到 object/另一 lambda 时外层 return 语义破坏"的问题。
- **局部返回 vs 非局部返回**：普通内联 lambda 中 `return` 是从**外层命名函数**返回；`crossinline`/非内联位置只能局部返回——面试常把二者说反。

### 【为什么这样设计】

内联把高阶抽象的运行时成本压到接近手写循环/回调注册；`noinline`/`crossinline` 是"对象身份"与"非局部控制流"两条无法两全时的显式选择，避免编译器猜业务意图。

### 【面试官视角：考察点 / 加分 / 减分】

- 考察点：inline 消除的是什么分配、crossinline 禁止什么、与高阶函数关系。
- 加分：提到公共 inline 过多导致调用点膨胀、编译时间与 ABI 的权衡；提到 `inline` 属性访问器。
- 减分：说 inline 一定更快/一定生成单份机器码不提调用点；把 crossinline 当"更强制的 noinline"。

### 【追问链】

1. 什么时候不该 inline？→ lambda 体量大、需作为对象传递、公共 API 慎重——包体积与编译时间。
2. inline 与 JIT 的关系？→ 编译期展开与 JVM JIT 内联互补；Kotlin inline 主要解决 lambda 分配与调用层。
3. 与 `value class` 包装函数类型有何不同？→ value class 仍有一次装箱边界（见下节）。

---

## 二、reified 与类型擦除绕过

### 问题

`reified` 为什么能让 `T::class.java` 在泛型处工作？它绕过了 JVM 类型擦除的哪一部分？

### 机制级答案

- **擦除现状**：JVM 泛型在字节码层擦除，运行时 `List<String>` 与 `List<Int>` 的类型参数不可直接取回；`T::class` 在普通泛型函数中无法拿到具体 `T` 的 Class。
- **reified**：仅允许与 **inline 函数**结合；类型参数在调用点被**特化实例化**（编译为具体类型或对应 Class 常量），函数体内可 `T::class`、`is T`、`as T`、`java.lang.String::class.java` 等。每次调用点为每个类型参数生成内联体（体膨胀）。
- **典型用途**：轻量序列化/Intent extras、类型安全的 DI/存储 API、`Json.decodeList<T>()` 类设计（完整序列化方案还需具体库，reified 只解决类型 token 的一部分）。
- **边界**：不能用于非 inline；不能用于类/接口的类型参数（只有函数）；不能用于 Java 互操作的泛型擦除点；数组/可空类型等细节按编译器规则。

### 【为什么这样设计】

擦除是 JVM 兼容性选择；reified 用"调用点展开"在不改 JVM 的前提下恢复类型 token——是**以代码膨胀换类型信息**，故强制与 inline 绑定，避免误用导致 ABI 膨胀。

### 【面试官视角：考察点 / 加分 / 减分】

- 考察点：擦除根因、reified 与 inline 绑定、`T::class` 失败场景。
- 加分：提到 `TypeToken`/`object : TypeReference<T>(){}` 匿名类保留泛型的替代；提到 Android 上 Intent/Bundle 封装常见用法。
- 减分：以为 reified 改变了 JVM 泛型；在非 inline 泛型里硬要 `T::class`。

### 【追问链】

1. Java 侧如何拿泛型？→ 反射 `getGenericSuperclass`/`ParameterizedType`，或 Guava TypeToken 思路。
2. reified 能用于 suspend 吗？→ 无特殊限制，仍受 suspend + inline 规则约束。
3. 性能影响？→ 调用点体积增大；热路径滥用影响 dex 与编译时间。

---

## 三、value class（内联类）：装箱与性能边界

### 问题

`value class` 如何避免包装对象分配？什么情况下仍会装箱？

### 机制级答案

- **语义**：`value class Foo(val s: String)`（曾用名 inline class）——单主构造属性；运行时尽量以底层类型表示，避免 `Foo` 对象分配；有独立类型（非别名），可实现接口。
- **mangling**：方法名可能带 name mangling 防止与底层类型签名冲突（对 Java 可见性不友好）；需要时用 `@JvmName` 等处理。
- **装箱边界（面试核心）**：
  1. **可空性**：`Foo?` 在可能为 null 的路径常需装箱（与普通值类型 null 表示冲突）；
  2. **泛型**：作为泛型类型参数 `T` 被擦除存储/装箱时（`List<Foo>` 元素），可能装箱；
  3. **接口实现**：值类装箱后才是完整对象（接口方法表）；
  4. **不安全操作/反射/跨边界**（部分场景由编译器插入防御性装箱）。
- **用途**：ID 类型（`UserId(String)`）防类型混用；度量单位；与裸 `String`/`Long` 混用事故的类型级防御。

### 【为什么这样设计】

业务里"同底层类型不同语义"大量存在；值类提供零成本（尽力）抽象，把运行时 bug 变成编译错误——装箱边界是 JVM 表示的真实约束，编译器在边界处诚实回退。

### 【面试官视角：考察点 / 加分 / 减分】

- 考察点：什么时候还是分配、与 data class 区别、可空与泛型陷阱。
- 加分：提到 name mangling 与 Java 互操作；提到与 `Any` 相等性基于底层值。
- 减分：宣称永远零分配；在需要可空集合元素的场景盲目用 value class 又说性能完美。

### 【追问链】

1. value class 能做 map key 吗？→ 可，基于值相等；注意装箱路径。
2. 与 typealias 差异？→ typealias 无新类型，编译期别名；value class 有独立类型检查。
3. 二进制兼容注意？→ mangling 与表示变化影响 Java 消费者。

---

## 四、委托属性：lazy、by map、自定义

### 问题

Kotlin 属性委托（`by`）的工作机制是什么？`lazy` 的线程安全模式有哪些？

### 机制级答案

- **机制**：`val x: T by expr` 要求 `expr` 提供 `getValue`（及 val 所需）/ `var` 另需 `setValue`——通过操作符约定（成员或扩展）。编译器生成的 `x` 访问转为对委托对象的调用；`property` 元数据可传给委托（`getValue(thisRef, property)`）。
- **by lazy**：
  - 默认 `LazyThreadSafetyMode.SYNCHRONIZED`：首次访问时初始化，锁保证只一次（双重检查在实现层）；
  - `PUBLICATION`：可多次初始化，首个写入者胜出（其他线程计算作废）——读侧无锁，初始化函数需幂等安全；
  - `NONE`：完全无同步，仅单线程/外部已同步场景；
  - 委托本身持有值，生命周期跟随宿主；不要把 lazy 当作用域副作用执行器（重复风险/锁顺序）。
- **by map / 委托到 MutableMap**：`val a: String by map`——常见于动态配置、 savedInstanceState、DI 容器取出（组件化路由/参数注入）；类型错误延迟到访问时（运行时）——便利与安全的交换。
- **自定义委托**：实现 `getValue/setValue` 或提供 `provideDelegate` 决定委托安装（校验 key、懒绑定）；`remember`/`viewModels` 等本质都是委托 + 生命周期持有。

### 【为什么这样设计】

委托把"横切样板"（懒加载、观察、存在性检查、生命周期感知）从继承/装饰器模式收敛到属性访问语法——单一访问点变更即可换策略；lazy 的三档线程模式对应不同并发预算。

### 【面试官视角：考察点 / 加分 / 减分】

- 考察点：委托协议、lazy 默认锁、NONE/PUBLICATION 适用、provideDelegate。
- 加分：指出 by map 运行时才炸；`viewModels()` 等是委托 + LifecycleOwner 作用域；lazy 与循环依赖/死锁。
- 减分：说委托是反射黑魔法；在多线程 UI 配置上用 `NONE`。

### 【追问链】

1. `provideDelegate` 何时用？→ 安装期校验/包装（如属性名 → 存储 key）。
2. lazy 初始化抛异常？→ 失败后状态按实现重置为未初始化，下次访问可能重试——异常处理需在 initializer 内消化。
3. 与 `by viewModels()` 关系？→ Activity KTX 委托，首次访问创建并缓存 ViewModel，作用域为 ViewModelStore。

---

## 五、扩展函数的静态解析与 this 优先级

### 问题

扩展函数是运行时多态吗？成员与扩展同名时如何选择？

### 机制级答案

- **静态解析**：扩展在**编译期**根据接收者静态类型绑定，不参与虚方法表多态。`fun View.hide()` 不可被"子类同签名扩展"重写；运行时对象是子类也调用编译期选中的扩展。
- **成员优先**：同名同签名时**成员胜出**；扩展签名不同则按重载规则。这造成"看似可扩展 API 实际走了成员"的排查陷阱。
- **作用域与导入**：扩展必须导入（或同包/伴生接收者）；接收者类型过泛（如 `Any`）会污染重载集——库代码慎用 `Any` 扩展。
- **this 优先级**：内层 receiver 的成员/扩展优先于外层；显式 `this@label` 消歧。多层 receiver（限定 this）是扩展 API 设计常见点。
- **扩展 vs 继承设计**：开放类用虚方法；封闭类/第三方类用扩展组合——但扩展不能访问私有状态，是"无侵入 API 面"不是"真成员"。

### 【为什么这样设计】

静态解析保证可预测与开销（无分发）；代价是运行时类型无法改变行为——扩展是"命名空间级组合"，不是子类型多态。

### 【面试官视角：考察点 / 加分 / 减分】

- 考察点：静态分派、成员优先、导入作用域、与 Java static import 类比（不完全等同）。
- 加分：提到扩展属性同样规则；提到 `with(receiver)` 内解析；提到与装饰器/代理模式边界。
- 减分：以为扩展可重写；在库内用 `Any.` 扩展导致用户重载冲突。

### 【追问链】

1. 扩展能访问私有成员？→ 不能（无特权）；有特权的是嵌套/同一文件受限场景。
2. 泛型扩展 `fun <T> List<T>.second()` 与成员冲突？→ 按重载与静态类型；仍成员优先。
3. Android 常见坑？→ 对 `Fragment.view` 空安全扩展与业务扩展同名误导。

---

## 六、契约 contracts（一句话深度）

### 问题

Kotlin contracts 解决什么问题？边界在哪里？

### 机制级答案

- **作用**：向编译器声明函数的效果，如 `implies(condition)`（调用后 condition 为真）、`returns(true) iff condition`——使调用点获得**智能转换/可达性**信息，例如自定义 `fun Any?.isNotNull(): Boolean = this != null` 配合 `contract { returns(true) implies (this@isNotNull != null) }` 后 `if (x.isNotNull()) x.length` 中 x 被智能转换。
- **限制**：稳定 API 覆盖有限（`check`/`require`/`also`/`run`/`let`/`takeIf` 等部分标准库）；用户自定义契约仍受限（实验性/固定语法形式）；契约不改变运行时行为，只影响编译器数据流分析。
- **意义**：把"文档级断言"升级为编译器可证明的前置/后置条件——空安全与控制流质量的关键。

### 【为什么这样设计】

智能转换需要编译器信任边界；契约用受限 DSL 让标准库与（部分）用户代码声明信任点，避免全量程序证明的复杂度。

### 【面试官视角：考察点 / 加分 / 减分】

- 考察点：契约 = 编译器提示非运行时魔法；与智能转换关系。
- 加分：提到 `returns`/`callsInPlace`（初始化上下文）；提到实验性演进以官方文档为准。
- 减分：以为契约可做运行时校验；以为任意函数都能写完整后置条件。

### 【追问链】

1. `callsInPlace` 干什么？→ 声明 lambda 以特定方式调用一次，使 `val` 可在 lambda 内初始化——`runCatching`/`also` 等基础。
2. 无契约时怎么办？→ 手动 `let`/本地 val 收窄类型。
3. 与 Java 互操作？→ Java 调用方无智能转换收益。

---

## 一、高阶函数与内联对性能的意义

### 问题

Kotlin 集合操作（`map`/`filter`/`fold`）每次调用分配中间对象吗？`inline` 如何改变这一点？

### 机制级答案

- **默认高阶集合 API 大多是 `inline`**：标准库中 `map`/`filter`/`forEach`/`fold` 等接受 lambda 的扩展几乎都标记 `inline`，调用点展开为循环体，**无 lambda 对象、无中间集合的额外函数调用层**（`map` 仍分配结果列表本身）。
- **中间集合**：链式 `list.map{}.filter{}.map{}` 在非序列 API 下产生**多个中间 List**——时间与分配随链长增长；这是"函数式链"的真实成本，与是否内联无关。
- **何时仍分配**：lambda 捕获外部 var/引用且需对象身份时；非 inline 高阶函数（第三方未标 inline、函数类型参数传递给非内联）；把 lambda 存变量再传（间接调用）。
- **工程意识**：列表热路径（RecyclerView 绑定、大列表转换）避免长链多次分配结果集；考虑 `Sequence`、就地循环、或一次遍历完成多变换。

### 【为什么这样设计】

内联高阶函数让"表达清晰的集合管线"在 JVM 上接近手写 for 的分配特征——Kotlin 把性能责任部分交给编译期变换，而不是要求用户放弃抽象。

### 【面试官视角：考察点 / 加分 / 减分】

- 考察点：inline 消除的是调用与 lambda 分配、中间 List 是另一问题。
- 加分：区分"内联收益"与"中间集合收益"两条优化线；提到公共 inline 对 dex 的影响。
- 减分：以为 map/filter 无分配；把慢全部归咎于没写 for 循环。

### 【追问链】

1. 为什么标准库能 inline 而用户函数可以吗？→ 用户高阶函数也可 `inline`，受同一约束（非局部返回、noinline）。
2. 装箱 Int 在 `List<Int>`？→ 泛型元素装箱——与 map 内联无关；值集合用专门类型/数组 API 意识。
3. 性能测量？→ 基准测试防 JIT 逃逸假象，不单看一次微基准。

---

## 二、Sequence：惰性求值边界

### 问题

`sequence {}` / `asSequence()` 与直接对 `List` 链式操作的本质差异？什么场景反而更慢？

### 机制级答案

- **冷序列语义**：Sequence 是惰性管道——每个操作符返回描述符，**终端操作（`toList`/`first`/`sum` 等）才驱动**逐元素穿过整条链；单遍消费，无中间集合。
- **与 List 链对比**：List 链 = 多遍、多中间集合；Sequence = 单遍、每元素走完链（除非 `take` 短路）。短路场景（`firstOrNull` 命中即停）Sequence 常显著更优。
- **何时更慢**：小集合 + 短链——iterator/lambda 虚调用与无法 JIT 优化的逐元素分发可能不敌一次 List 中间分配；**默认对小列表直接用 List 链或循环**。
- **`sequence {}` 构建器**：`yield`/`yieldAll` 手写惰性生产者；注意不要在冷序列里做每元素昂贵副作用且多次 collect——**每次终端操作重新执行**（冷流再生产，与 Flow 冷流同构）。
- **与 Flow 关系**：Flow 是协程版惰性流（挂起、上下文、取消感知）；Sequence 是同步、无取消——概念上"冷"一致。

### 【为什么这样设计】

惰性 + 单遍是对"集合即全量内存"的补充：流式处理与短路在 O(1) 额外内存下表达；代价是每元素间接层，故 API 层提供两种形态让用户按规模选择。

### 【面试官视角：考察点 / 加分 / 减分】

- 考察点：惰性触发时机、短路收益、小集合陷阱、冷重放。
- 加分：给出"多大/多链再用 Sequence"的经验边界（如链长或元素量明显时）；类比 Flow 冷流。
- 减分：所有集合都 `.asSequence()`；不知道终端操作才执行。

### 【追问链】

1. Sequence 能并发吗？→ 标准序列同步单线程模型；并行用集合并行 API/协程分发，不是 asSequence。
2. 多次 toList 会怎样？→ 冷序列重新生产。
3. 与 Java Stream？→ 概念类似惰性管线；Sequence 挂起协同用协程，Stream 用 spliterator/并行池。

---

## 三、协变 / 逆变：声明位的实际影响

### 问题

`out` / `in` 在 Kotlin 中解决什么？写错为什么会在调用点炸出 Java 风格通配符问题？

### 机制级答案

- **不变默认**：`class Box<T>` 不可用 `Box<String>` 赋给 `Box<Any>`（写入安全破坏）。Java 用 `? extends/super` 在使用点补救；Kotlin 用 `out`/`in` 在**声明位**标注生产/消费位置。
- **out（协变）**：`interface Source<out T> { fun get(): T }` —— 只出现在"输出"位置（返回值）；`Source<String>` 是 `Source<Any>` 的子类型；不能有 `fun set(t: T)` 类消费方法。
- **in（逆变）**：`interface Sink<in T> { fun put(t: T) }` —— 只作输入；`Sink<Any>` 是 `Sink<String>` 的子类型。
- **声明处型变收益**：调用方免写通配符；函数接口常用 `Function1<in P, out R>` 对应 `P.() -> R` 函数类型——函数类型 variance 直接影响 lambda 传参。
- **平台声明便利**：`@JvmSuppressWildcards` 等处理 Java 通配符生成——库作者 API 设计必考点（见互操作）。

### 【为什么这样设计】

声明位型变把方差检查前移到类型定义，一次声明全体调用方获益；与 Java 使用点通配符相比更严也更省心——代价是设计接口时就要想清数据流向。

### 【面试官视角：考察点 / 加分 / 减分】

- 考察点：out/in 与生产-消费、函数类型方差、与 Java extends 对应。
- 加分：提到 `List` 是 `List<out E>` 协变、`MutableList` 不变；提到 star projection `*`。
- 减分：到处 `out Any` 胡标；说不清为何可写方法不能出现在 out 位置。

### 【追问链】

1. 为什么 MutableList 不能协变？→ 有 `set`/`add`，写入需要 in 位置。
2. star projection？→ `Box<*>` 读出 `Any?`、写入拒绝——对未知参数的保守访问。
3. Java 泛型与 Kotlin 混用？→ 通配符进 Kotlin 变平台类型方差，见下节。

---

## 四、与 Java 互操作十大坑

### 问题

生产中 Kotlin/Java 混合工程有哪些高频互操作陷阱？如何规避？

### 机制级答案

1. **默认参数与 @JvmOverloads**：Kotlin 重载默认值生成合成；Java 侧只见完整签名或 `@JvmOverloads` 生成的重载链——**库 API 要么 @JvmOverloads 要么接受 Java 调用方必须传全参**。
2. **静态成员**：伴生对象方法默认 `Companion.method()`；加 `@JvmStatic`；`@JvmField` 暴露字段；`const val` 才是真正的编译期常量。
3. **@JvmName / @JvmOverloads / @JvmStatic / @JvmField**：互操作注解族——库代码 review 清单。
4. **空平台类型**：Java 无注解返回值 → `T!`；Kotlin 调用不检查 → NPE。规则：**封装边界立刻收敛为可空/非空并跑测试**。
5. **泛型与通配符**：Kotlin `List<out E>` 对 Java 是 `List<? extends E>`；Java 实现/继承处需 `@JvmSuppressWildcards` 减少恼人 `? extends` 展开——API 设计期决定。
6. **默认方法与接口**：Kotlin 接口默认实现编译策略与 Java 8 default 不同细节；混继承注意二进制。
7. **字符串/字符**：Java `char`/`String` 与 Kotlin `Char`/`String`；`@JvmStatic` 扩展在伴生。
8. **可见性**：Kotlin `internal` 对 Java 是 public（名字可能 mangling）——**不要把 internal 当安全边界**。
9. **SAM 转换**：Kotlin lambda → Java 函数式接口可自动 SAM；Kotlin 接口需 `fun interface`。
10. **异常与检查异常**：Kotlin 无受检异常；Java 侧 `throws` 对 Kotlin 调用不强制，反向声明不产生检查。

### 【为什么这样设计】

互操作层是"语言语义 vs JVM 现实"的翻译层；十大坑几乎全是**编译器默认选择对 Java 最不惊讶**或**对 Kotlin 最符合语言习惯**之间的裂缝——库作者用注解桥接，应用层在边界显式收窄。

### 【面试官视角：考察点 / 加分 / 减分】

- 考察点：@Jvm* 注解族、平台类型、internal 的 Java 可见性、SAM。
- 加分：能给出"库模块 API 评审检查单"；提到 abi 工具/`apiDump`（binary-compatibility-validator）思路。
- 减分：混用模块从不加注解；internal 用来藏敏感 API。

### 【追问链】

1. 如何写"对 Java 友好的 Kotlin API"？→ @JvmStatic/@JvmOverloads/@JvmName + 空注解 + 避免默认参数组合爆炸。
2. Java 实现 Kotlin 接口默认方法？→ 按编译目标 JVM 版本与接口规则，测跨语言继承用例。
3. internal 的真实边界？→ 同模块编译期；运行时与 Java public 无强制隔离。
