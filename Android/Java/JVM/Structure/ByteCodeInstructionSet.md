# 字节码指令集简介

Java 虚拟机的指令由一个字节长度的、代表着某种特定操作含义的操作码（Opcode）以及跟随其后的零至多个代表此操作所需参数的操作数（ Operands）所构成。虚拟机中许多指令并不包含操作数，只有一个操作码。如果忽略异常处理，那 Java 虚拟机的解释器使用下面这个伪代码的循环即可有效地工作：

```java
do {

自动计算 PC 寄存器以及从 PC 寄存器的位置取出操作码;

if (存在操作数) 取出操作数;

执行操作码所定义的操作

} while (处理下一次循环);
```

操作数的数量以及长度取决于操作码，如果一个操作数的长度超过了一个字节，那它将会以Big-Endian 顺序存储——即高位在前的字节序。举个例子，如果要将一个 16 位长度的无符号整数使用两个无符号字节存储起来（将它们命名为 byte1 和 byte2），那它们的值应该是这样的：

```java
(byte1 << 8) | byte2
```

字节码指令流应当都是单字节对齐的，只有“ tableswitch”和“ lookupswitch”两条指
令例外，由于它们的操作数比较特殊，都是以 4 字节为界划分开的，所以这两条指令那个也需要预留出相应的空位来实现对齐。
限制 Java 虚拟机操作码的长度为一个字节，并且放弃了编译后代码的参数长度对齐，是为了尽可能地获得短小精干的编译代码，即使这可能会让 Java 虚拟机的具体实现付出一定的性能成本为代价。由于每个操作码只能有一个字节长度，所以直接限制了整个指令集的数量①，又由于没有假设数据是对齐好的，这就意味着虚拟机处理那些超过一个字节的数据的时候，不得不在运行时从字节中重建出具体数据的结构，这在某种程度上会损失一些性能。

## 数据类型与 Java 虚拟机

在 Java 虚拟机的指令集中，大多数的指令都包含了其操作所对应的数据类型信息。举个例子，iload 指令用于从局部变量表中加载 int 型的数据到操作数栈中，而 fload 指令加载的则是float 类型的数据。这两条指令的操作可能会是由同一段代码来实现的，但它们必须拥有各自独立的操作符。
对于大部分为与数据类型相关的字节码指令，他们的操作码助记符中都有特殊的字符来表明专门为哪种数据类型服务：i 代表对 int 类型的数据操作，l 代表 long，s 代表 short，b 代表 byte，c 代表 char， f 代表 float， d 代表 double， a 代表reference。也有一些指令的助记符中没有明确的指明操作类型的字母，例如arraylength 指令，它没有代表数据类型的特殊字符，但操作数永远只能是一个数组类型的对象。还有另外一些指令，例如无条件跳转指令 goto 则是与数据类型无关的。
由于 Java 虚拟机的操作码长度只有一个字节，所以包含了数据类型的操作码对指令集的设计带来了很大的压力：如果每一种与数据类型相关的指令都支持 Java 虚拟机所有运行时数据类型的话，那恐怕就会超出一个字节所能表示的数量范围了。因此， Java 虚拟机的指令集对于特定的操作只提供了有限的类型相关指令去支持它，换句话说，指令集将会故意被设计成非完全独立的（ Not Orthogonal，即并非每种数据类型和每一种操作都有对应的指令）。有一些单独的指令可以在必要的时候用来将一些不支持的类型转换为可被支持的类型。
表 2.2 列举了 Java 虚拟机所支持的字节码指令集，通过使用数据类型列所代表的特殊字符替换 opcode 列的指令模板中的 T，就可以得到一个具体的字节码指令。如果在表中指令模板与数据类型两列共同确定的格为空，则说明虚拟机不支持对这种数据类型执行这项操作。例如 load 指令有操作 int 类型的 iload，但是没有操作 byte 类型的同类指令。

请注意，从表 2.2 中看来，大部分的指令都没有支持整数类型 byte、 char 和 short，甚至没有任何指令支持 boolean 类型。编译器会在编译期或运行期会将 byte 和 short 类型的数据带符号扩展（ Sign-Extend）为相应的 int 类型数据，将 boolean 和 char 类型数据零位扩展（ Zero-Extend）为相应的 int 类型数据。与之类似的，在处理 boolean、 byte、 short 和char 类型的数组时，也会转换为使用对应的 int 类型的字节码指令来处理。因此，大多数对于boolean、 byte、 short 和 char 类型数据的操作，实际上都是使用相应的对 int 类型作为运算类型（ Computational Type）。

表 2.2 Java 虚拟机指令集所支持的数据类型

|   opcode   |  byte   |  short  |    int     |  long   |  float  | double  |  char   | reference  |
| :--------: | :-----: | :-----: | :--------: | :-----: | :-----: | :-----: | :-----: | :--------: |
|   Tipush   | bipush  | sipush  |            |         |         |         |         |            |
|   Tconst   |         |         |   iconst   | lconst  | fconst  | dconst  |         |   aconst   |
|   Tload    |         |         |   iload    | llload  |  fload  |  dload  |         |   aload    |
|   Tstore   |         |         |   istore   | lstore  | fstore  | dstore  |         |   astore   |
|    Tinc    |         |         |    iinc    |         |         |         |         |            |
|   Taload   | baload  | saload  |   iaload   | laload  | faload  | daload  | caload  |   aaload   |
|  Tastore   | bastore | sastore |  iastore   | lastore | fastore | dastore | castore |  aastore   |
|    Tadd    |         |         |    iadd    |  ladd   |  fadd   |  dadd   |         |            |
|    Tsub    |         |         |    isub    |  lsub   |  fsub   |  dsub   |         |            |
|    Tmul    |         |         |    imul    |  lmul   |  fmul   |  dmul   |         |            |
|    Tdiv    |         |         |    idiv    |  ldiv   |  fdiv   |  ddiv   |         |            |
|    Trem    |         |         |    irem    |  lrem   |  frem   |  drem   |         |            |
|    Tneg    |         |         |    ineg    |  lneg   |  fneg   |  dneg   |         |            |
|    Tshl    |         |         |    ishl    |  lshl   |         |         |         |            |
|    Tshr    |         |         |    ishr    |  lshr   |         |         |         |            |
|   Tushr    |         |         |   iushr    |  lushr  |         |         |         |            |
|    Tand    |         |         |    iand    |  land   |         |         |         |            |
|    Tor     |         |         |    ior     |   lor   |         |         |         |            |
|    Txor    |         |         |    ixor    |  lxor   |         |         |         |            |
|    i2T     |   i2b   |   i2s   |            |   i2l   |   i2f   |   i2d   |         |            |
|    l2T     |         |         |    l2i     |         |   l2f   |   l2d   |         |            |
|    f2T     |         |         |    f2i     |   f2l   |         |   f2d   |         |            |
|    d2T     |         |         |    d2i     |   d2l   |   d2f   |         |         |            |
|    Tcmp    |         |         |            |  lcmp   |         |         |         |            |
|   Tcmpl    |         |         |            |         |  fcmpl  |  dcmpl  |         |            |
|   Tcmpg    |         |         |            |         |  fcmpg  |  dcmpg  |         |            |
| if_Tcmp OP |         |         | if_icmp OP |         |         |         |         | if_acmp OP |
|  Treturn   |         |         |  ireturn   | lreturn | freturn | dreturn |         |  areturn   |


在 Java 虚拟机中，实际类型与运算类型之间的映射关系，如表 2.3 所示。
表 **2.3 Java **虚拟机指令集所支持的数据类型

|     实际类型      |     运算类型      |  分类  |
| :-----------: | :-----------: | :--: |
|    boolean    |      int      | 分类一  |
|     byte      |      int      | 分类一  |
|     char      |      int      | 分类一  |
|     short     |      int      | 分类一  |
|      int      |      int      | 分类一  |
|     float     |     float     | 分类一  |
|   reference   |   reference   | 分类一  |
| returnAddress | returnAddress | 分类一  |
|     long      |     long      | 分类二  |
|    double     |    double     | 分类二  |


有部分对操作栈进行操作的 Java 虚拟机指令（例如 pop 和 swap 指令）是与具体类型无关的，不过这些指令也必须受到运算类型分类的限制，这些分类也在表 2.3 中列出了。

## 加载和存储指令

加载和存储指令用于将数据从栈帧的局部变量表和操作数栈之间来回传输：

* 将一个局部变量加载到操作栈的指令包括有： iload、iload_<n>、lload、lload_<n>、fload、 fload_<n>、 dload、 dload_<n>、 aload、 aload_<n>
* 将一个数值从操作数栈存储到局部变量表的指令包括有： istore、 istore_<n>、lstore、 lstore_<n>、 fstore、 fstore_<n>、 dstore、 dstore_<n>、 astore、astore_<n>
* 将一个常量加载到操作数栈的指令包括有： bipush、 sipush、 ldc、 ldc_w、 ldc2_w、aconst_null、iconst_m1、iconst_<i>、lconst_<l>、fconst_<f>、dconst_<d>
* 扩充局部变量表的访问索引的指令： wide访问对象的字段或数组元素的指令也同样会与操作数栈传输数据。

上面所列举的指令助记符中，有一部分是以尖括号结尾的（例如 iload_<n>），这些指令助记符实际上是代表了一组指令（例如 iload_<n>，它代表了 iload_0、 iload_1、 iload_2 和 iload_3 这几条指令）。这几组指令都是某个带有一个操作数的通用指令（例如 iload）的特殊形式，对于这若干组特殊指令来说，它们表面上没有操作数，不需要进行取操作数的动作，但操作数都是在指令中隐含的。除此之外，他们的语义与原生的通用指令完全一致（例如 iload_0 的语义与操作数为 0 时的 iload 指令语义完全一致）。在尖括号之间的字母制定了指令隐含操作数的数据类型， <i>代表是 int 形数据， <l>代表 long 型， <f>代表 float 型， <d>代表 double型。在操作 byte、 char 和 short 类型数据时，也用 int 类型表示。
这种指令表示方法，在整个《 Java 虚拟机规范》 之中都是通用的。

## 运算指令

算术指令用于对两个操作数栈上的值进行某种特定运算，并把结果重新存入到操作栈顶。大体上运算指令可以分为两种：对整型数据进行运算的指令与对浮点型数据进行运算的指令，无论是那种算术指令，都是使用 Java 虚拟机的数字类型的。数据没有直接支持 byte、 short、 char 和boolean 类型的算术指令，对于这些数据的运算，都是使用操作 int 类型的指令。整数与浮点数的算术指令在溢出和被零除的时候也有各自不同的行为，所有的算术指令包括：

* 加法指令： iadd、 ladd、 fadd、 dadd
* 减法指令： isub、 lsub、 fsub、 dsub
* 乘法指令： imul、 lmul、 fmul、 dmul
* 除法指令： idiv、 ldiv、 fdiv、 ddiv
* 求余指令： irem、 lrem、 frem、 drem
* 取反指令： ineg、 lneg、 fneg、 dneg
* 位移指令： ishl、 ishr、 iushr、 lshl、 lshr、 lushr
* 按位或指令： ior、 lor
* 按位与指令： iand、 land
* 按位异或指令： ixor、 lxor
* 局部变量自增指令： iinc
* 比较指令： dcmpg、 dcmpl、 fcmpg、 fcmpl、 lcmp

Java 虚拟机的指令集直接支持了在《 Java 语言规范》 中描述的各种对整数及浮点数操作的语义。
Java 虚拟机没有明确规定整型数据溢出的情况，但是规定了在处理整型数据时，只有除法指令（ idiv 和 ldiv）以及求余指令（ irem 和 lrem）出现除数为零时会导致虚拟机抛出异常，如果发生了这种情况，虚拟机将会抛出 ArithmeitcException 异常。
Java 虚拟机在处理浮点数时，必须遵循 IEEE 754 规范中所规定行为限制。也就是说 Java虚拟机要求完全支持 IEEE 754 中定义的非正规浮点数值（ Denormalized Floating-Point Numbers）和逐级下溢（ Gradual Underflow）。这些特征将会使得某些数值算法处理起来变得更容易一些。
Java 虚拟机要求在进行浮点数运算时，所有的运算结果都必须舍入到适当的进度，非精确的结果必须舍入为可被表示的最接近的精确值，如果有两种可表示的形式与该值一样接近，那将优先选择最低有效位为零的。这种舍入模式也是 IEEE 754 规范中的默认舍入模式，称为向最接近数舍入模式。

在把浮点数转换为整数时， Java 虚拟机使用 IEEE 754 标准中的向零舍入模式，这种模式的舍入结果会导致数字被截断，所有小数部分的有效字节都会被丢弃掉。 向零舍入模式将在目标数值类型中选择一个最接近，但是不大于原值的数字来作为最精确的舍入结果。

Java 虚拟机在处理浮点数运算时，不会抛出任何运行时异常（这里所讲的是 Java 的异常，请勿与 IEEE 754 规范中的浮点异常互相混淆），当一个操作产生溢出时，将会使用有符号的无穷大来表示，如果某个操作结果没有明确的数学定义的话，将会时候 NaN 值来表示。所有使用 NaN值作为操作数的算术操作，结果都会返回 NaN。
在对 long 类型数值进行比较时，虚拟机采用带符号的比较方式，而对浮点数值进行比较时（ dcmpg、 dcmpl、 fcmpg、 fcmpl），虚拟机采用 IEEE 754 规范说定义的无信号比较（ Nonsignaling Comparisons）方式。

## 类型转换指令

类型转换指令可以将两种 Java 虚拟机数值类型进行相互转换，这些转换操作一般用于实现用户代码的显式类型转换操作，或者用来处理 Java 虚拟机字节码指令集中指令非完全独立独立的问题。
Java 虚拟机直接支持（译者注：“直接支持”意味着转换时无需显式的转换指令）以下数值的宽化类型转换（ Widening Numeric Conversions，小范围类型向大范围类型的安全转换）：

* int 类型到 long、 float 或者 double 类型
* long 类型到 float、 double 类型
* float 类型到 double 类型

窄化类型转换（ Narrowing Numeric Conversions）指令包括有： i2b、 i2c、 i2s、 l2i、f2i、 f2l、 d2i、 d2l 和 d2f。窄化类型转换可能会导致转换结果产生不同的正负号、不同的数量级，转换过程很可能会导致数值丢失精度。
在将 int 或 long 类型窄化转换为整数类型 T 的时候，转换过程仅仅是简单的丢弃除最低位N 个字节以外的内容， N 是类型 T 的数据类型长度，这将可能导致转换结果与输入值有不同的正负号（译者注：在高位字节符号位被丢弃了）。



在将一个浮点值转窄化转换为整数类型 T（ T 限于 int 或 long 类型之一） 的时候，将遵循以
下转换规则：
 如果浮点值是 NaN，那转换结果就是 int 或 long 类型的 0
 否则，如果浮点值不是无穷大的话，浮点值使用 IEEE 754 的向零舍入模式（ §2.8.1）
取整，获得整数值 v，这时候可能有两种情况：
 如果 T 是 long 类型，并且转换结果在 long 类型的表示范围之内，那就转换为 long
类型数值 v
 如果 T 是 int 类型，并且转换结果在 int 类型的表示范围之内，那就转换为 int
类型数值 v
 否则：
 如果转换结果 v 的值太小（包括足够小的负数以及负无穷大的情况），无法使用 T 类
型表示的话，那转换结果取 int 或 long 类型所能表示的最小数字。
 如果转换结果 v 的值太大（包括足够大的正数以及正无穷大的情况），无法使用 T 类
型表示的话，那转换结果取 int 或 long 类型所能表示的最大数字。
从 double 类型到 float 类型做窄化转换的过程与 IEEE 754 中定义的一致，通过 IEEE 754
向最接近数舍入模式（ §2.8.1）舍入得到一个可以使用 float 类型表示的数字。如果转换结果
的绝对值太小无法使用 float 来表示的话，将返回 float 类型的正负零。 如果转换结果的绝对值
太大无法使用 float 来表示的话，将返回 float 类型的正负无穷大，对于 double 类型的 NaN
值将就规定转换为 float 类型的 NaN 值。
尽管可能发生上限溢出、下限溢出和精度丢失等情况，但是 Java 虚拟机中数值类型的窄化转
换永远不可能导致虚拟机抛出运行时异常（此处的异常是指《 Java 虚拟机规范》 中定义的异常，
请读者不要与 IEEE 754 中定义的浮点异常信号产生混淆）。

## 对象创建与操作

虽然类实例和数组都是对象，但 Java 虚拟机对类实例和数组的创建与操作使用了不同的字节码指令：

* 创建类实例的指令： new
* 创建数组的指令： newarray， anewarray， multianewarray
* 访问类字段（ static 字段，或者称为类变量）和实例字段（非 static 字段，或者成为
  实例变量）的指令： getfield、 putfield、 getstatic、 putstatic
* 把一个数组元素加载到操作数栈的指令： baload、caload、saload、iaload、laload、
  faload、 daload、 aaload
* 将一个操作数栈的值储存到数组元素中的指令： bastore、 castore、 sastore、
  iastore、 fastore、 dastore、 aastore
* 取数组长度的指令： arraylength
* 检查类实例类型的指令： instanceof、 checkcast

## 操作数栈管理指令

Java 虚拟机提供了一些用于直接操作操作数栈的指令，包括： pop、 pop2、 dup、 dup2、dup_x1、 dup2_x1、 dup_x2、 dup2_x2 和 swap。

## 控制转移指令

控制转移指令可以让 Java 虚拟机有条件或无条件地从指定指令而不是控制转移指令的下一条指令继续执行程序。 控制转移指令包括有：

* 条件分支： ifeq、iflt、ifle、ifne、ifgt、ifge、ifnull、ifnonnull、if_icmpeq、if_icmpne、 if_icmplt, if_icmpgt、 if_icmple、 if_icmpge、 if_acmpeq 和if_acmpne。
* 复合条件分支： tableswitch、 lookupswitch
* 无条件分支： goto、 goto_w、 jsr、 jsr_w、 ret

在 Java 虚拟机中有专门的指令集用来处理 int 和 reference 类型的条件分支比较操作，为了可以无需明显标识一个实体值是否 null，也有专门的指令用来检测 null 值。
boolean 类型、 byte 类型、 char 类型和 short 类型的条件分支比较操作，都使用 int 类型的比较指令来完成，而对于 long 类型、 float 类型和 double 类型的条件分支比较操作， 则会先执行相应类型的比较运算指令（ §2.11.3），运算指令会返回一个整形值到操作数栈中，随后再执行 int 类型的条件分支比较操作来完成整个分支跳转。由于各种类型的比较最终都会转化为 int 类型的比较操作，基于 int 类型比较的这种重要性， Java 虚拟机提供了非常丰富的 int类型的条件分支指令。
所有 int 类型的条件分支转移指令进行的都是有符号的比较操作。

## 方法调用和返回指令

以下四条指令用于方法调用：
invokevirtual 指令用于调用对象的实例方法，根据对象的实际类型进行分派（虚方法分派），这也是 Java 语言中最常见的方法分派方式。
invokeinterface 指令用于调用接口方法，它会在运行时搜索一个实现了这个接口方法的对象，找出适合的方法进行调用。
invokespecial 指令用于调用一些需要特殊处理的实例方法，包括实例初始化方法、私有方法和父类方法。
invokestatic 指令用于调用类方法（ static 方法）。
而方法返回指令则是根据返回值的类型区分的，包括有 ireturn（当返回值是 boolean、byte、 char、 short 和 int 类型时使用）、 lreturn、 freturn、 dreturn 和 areturn，另外还有一条 return 指令供声明为 void 的方法、实例初始化方法、类和接口的类初始化方法使用。

## 抛出异常

在程序中显式抛出异常的操作会由 athrow 指令实现，除了这种情况，还有别的异常会在其他 Java 虚拟机指令检测到异常状况时由虚拟机自动抛出。

## 同步

Java 虚拟机可以支持方法级的同步和方法内部一段指令序列的同步，这两种同步结构都是使用管程（ Monitor）来支持的。

方法级的同步是隐式，即无需通过字节码指令来控制的，它实现在方法调用和返回操作之中。虚拟机可以从方法常量池中的方法表结构（ method_info Structure）中的 ACC_SYNCHRONIZED 访问标志区分一个方法是否同步方法。当方法调用时，调用指令将会检查方法的 ACC_SYNCHRONIZED 访问标志是否被设置，如果设置了，执行线程将先持有管程，然后再执行方法，最后再方法完成（无论是正常完成还是非正常完成）时释放管程。在方法执行期间，执行线程持有了管程，其他任何线程都无法再获得同一个管程。如果一个同步方法执行期间抛出了异常，并且在方法内部无法处理此异常，那这个同步方法所持有的管程将在异常抛到同步方法之外时自动释放。


同步一段指令集序列通常是由 Java 语言中的 synchronized 块来表示的， Java 虚拟机的指令集中有 monitorenter 和 monitorexit 两条指令来支持 synchronized 关键字的语义，正确实现 synchronized 关键字需要编译器与 Java 虚拟机两者协作支持。
结构化锁定（ Structured Locking）是指在方法调用期间每一个管程退出都与前面的管程进入相匹配的情形。因为无法保证所有提交给 Java 虚拟机执行的代码都满足结构化锁定，所以Java 虚拟机允许（但不强制要求）通过以下两条规则来保证结构化锁定成立。假设 T 代表一条线程， M 代表一个管程的话：
1. T 在方法执行时持有管程 M 的次数必须与 T 在方法完成（包括正常和非正常完成）时释放管程 M 的次数相等。
2. 找方法调用过程中，任何时刻都不会出现线程 T 释放管程 M 的次数比 T 持有管程 M 次数多的情况。
   请注意，在同步方法调用时自动持有和释放管程的过程也被认为是在方法调用期间发生。





































































