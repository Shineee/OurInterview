# 常量、 局部变量的使用和控制结构 

Java 虚拟机代码中展示了 Java 虚拟机设计和使用所遵循的一些通用特性。从第一个例子我们就可以感受到许多这类特性，示例如下：
spin()是一个很简单的方法，它进行了 100 次空循环：

```java
void spin() {

int i;

for (i = 0; i < 100; i++) {

; // Loop body is empty

}

}
```

编译后代码如下：

```java
Method void spin()

0 iconst_0 // Push int constant 0

1 istore_1 // Store into local variable 1 (i=0)

2 goto 8 // First time through don’t increment

5 iinc 1 1 // Increment local variable 1 by 1 (i++)

8 iload_1 // Push local variable 1 (i)

9 bipush 100 // Push int constant 100

11 if_icmplt 5 // Compare and loop if less than (i < 100) 
  
14 return // Return void when done
```

Java 虚拟机是基于栈架构设计的， 它的大多数操作是从当前栈帧的操作数栈取出 1 个或多个操作数，或将结果压入操作数栈中。每调用一个方法，都会创建一个新的栈帧，并创建对应方法所需的操作数栈和局部变量表（ 参见栈帧）。每条线程在运行时的任意时刻， 都会包含若干个由不同方法嵌套调用而产生的栈帧，当然也包括了若干个栈帧内部的操作数栈， 但是只有当前栈帧中的操作数栈才是活动的。
Java 虚拟机指令集合使用不同的字节码来区分不同的操作数类型，用于操作各种类型的变量。在 spin()方法中，只有对 int 类型的运算。 因此在编译代码里面，所采用的对类型数据进行操作的指令（ iconst_0、 istore_1、 iinc、 iload_1、 if_icmplt） 都是针对 int 型的。
spin()方法中， 0 和 100 两个常量分别使用了两条不同的指令压入操作数栈。对于 0 采用了iconst_0 指令， 它属于 iconst_<i>指令族。 而对于 100 采用 bipush 指令，这条指令会获取它的立即操作数（ Immediate Operand） ①压入到操作数栈中。
Java虚拟机经常利用操作码隐式包含某些操作数，譬如指令iconst<i> 中的i表示的int常量 -1、 0、 1、 2、 3、 4、 5。 iconst0 表示把 int 型的 0 值压入操作数栈，这样， iconst0不需要专门为入栈操作保存一个立即操作数的值，这样也避免了操作数的读取和解析（ Decode）的步骤。在本例中，把 0 压入操作数栈这个操作的指令由 iconst_0 改为 bipush 0 也能获取正确的结果，但是 spin()的编译代码会因此额外增加 1 个字节的长度。简单实现的虚拟机可能在每次循环时消耗更多的时间用于获取和解析这个操作数。因此使用隐式操作数可让编译后的代码更简洁，更高效。
spin()方法中， int 型的 i 被保存在编号为 1 的局部变量中②。因为大部分 Java 虚拟机指令操作的值是来源于操作数栈中出栈的值，而不是操作局部变量本身，所以在 Java 虚拟机的已编译代码中，在局部变量表和操作数栈之间传输值的指令很常见的， 在指令集里，这类操作也有特殊地支持。 spin()方法的第一个局部变量的传输过程由 istore_1 和 iload_1 指令完成，这两条指令都隐式指明了是对于第一个局部变量进行操作。 istore_1 指令作用是从操作数栈中弹出一个 int 型的值，并保存在第一个局部变量中。 iload_1 指令作用是将第一个局部变量的值压入操
作数栈。 

如何使用（以及重用） 局部变量由编译器的开发者所决定。 尤其是对于 load 和 store 指令，编译器的开发者应尽可能多的重用局部变量，这样会使得代码更高效，更简洁， 占用的内存（ 当前栈帧的空间） 更少。
某些对局部变量频繁进行的操作在 Java 虚拟机中也有特别地支持。 iinc 指令的作用是对局部变量加上一个长度为 1 字节有符号的递增量。 譬如 spin()方法中 iinc 指令，它的作用是对第一个局部变量（第一个操作数）的值增加 1（第二个操作数）。 iinc 指令很适合实现循环结构。
spin()方法的循环部分由这些指令完成：

```java
5 iinc 1 1 // Increment local 1 by 1 (i++)

8 iload_1 // Push local variable 1 (i)

9 bipush 100 // Push int constant 100

11 if_icmplt 5 // Compare and loop if less than (i < 100)

```

bipush 指令将 int 型的 100 压入操作数栈，然后 if_icmplt 指令将 100 从操作数栈中弹出值并与 i 进行比较，如果满足条件（即 i 的值小于 100），将转移到索引为 5 的指令继续执行，开始下一轮循环的迭代。否则， 程序将执行 if_icmplt 的下一条指令，即 return 指令。
如果在 spin()例子的中循环的计数器使用了非 int 类型，那么编译代码也要有调整。 譬如在 spin 例子中使用 double 型取代 int，则：

```java
void dspin() {

double i;

for (i = 0.0; i < 100.0; i++) {

; // Loop body is empty

}

}
```

编译后代码如下：

```java
Method void dspin()

0 dconst_0 // Push double constant 0.0

1 dstore_1 // Store into local variables 1 and 2

2 goto 9 // First time through don’t increment

5 dload_1 // Push local variables 1 and 2

6 dconst_1 // Push double constant 1.0

7 dadd // Add; there is no dinc instruction

8 dstore_1 // Store result in local variables 1 and 2

9 dload_1 // Push local variables 1 and 2

10 ldc2_w #4 // Push double constant 100.0

13 dcmpg // There is no if_dcmplt instruction

14 iflt 5 // Compare and loop if less than (i < 100.0) 

17 return // Return void when done
```

操作数据类型的指令已变成针对 double 类型值的指令了。（ ldc2_w 指令将在本章节晚些时候讨论）。
前文提到过 double 类型的值占用两个局部变量的空间，但是只能通过两个局部变量空间中索引较小的一个进行访问（ 这种情况对于 long 类型也一样）。譬如下面例子展示了 double 类型值的访问：

```java
double doubleLocals(double d1, double d2) {

return d1 + d2;

}
```

编译后代码如下：

```java
Method double doubleLocals(double,double)

0 dload_1 // First argument in local variables 1 and 2

1 dload_3 // Second argument in local variables 3 and 4

2 dadd

3 dreturn

```

注意到局部变量表中使用了一对局部变量来存储 doubleLocals()方法中的 double 值，这对局部变量不能被分开来进行单个操作。
Java 虚拟机中，操作码长度为 1 个字节，这使得编译后的代码显得很紧凑。但是同样意味着Java 虚拟机指令集必须保持一个较小的数量（小于 256 条， 1 字节所能表示的范围）。作为妥协，Java 虚拟机无法对全部操作都一视同仁地提供所有数据类型的支持。换句话说，并非每条指令都是完全独立（ Not Completely Orthogonal）支持一种类型的（参见表 2.2 “ Java 虚拟机指令集所支持的数据类型”）。
举例来说，在 spin()方法的 for 循环语句中，对于 int 型值的判断可以统一用 if_icmplt指令实现；但是，在 Java 虚拟机指令集合中，对于 double 类型的值的没有这样的指令。所以，在 dspin()方法中，对于 double 类型的值的操作就必须在 dcmpg 指令后面再联合 iflt 指令来实现。
Java 虚拟机支持下， 对 int 类型的数据的大部分操作可以直接进行。这在一定程度上是考虑到了 Java 虚拟机操作数栈和局部变量表的实现效率。当然也有考虑到了大多数程序都会对 int型数据进行频繁操作的原因。 Java 虚拟机对其余的数据类型的直接操作的支持较少，在 Java 虚拟机指令集中，没有对 byte， char 和 short 类型变量的 store、 load 和 add 等指令。 譬如，用 short 类型来实现 spin()中的循环时： 

```java
void sspin() {

short i;

for (i = 0; i < 100; i++) {

; // Loop body is empty

}

}

```

Java 虚拟机编译时要将其他可安全转化为 int 类型的数据转换为 int 类型进行操作。在将short 类型值转换为 int 类型值时，可以保证 short 类型值操作后结果一定在 int 类型的精度范围之内，因此 sspin()的编译后代码如下：

```java
Method void sspin()

0 iconst_0

1 istore_1

2 goto 10

5 iload_1 // The short is treated as though an int

6 iconst_1

7 iadd

8 i2s // Truncate int to short

9 istore_1

10 iload_1

11 bipush 100

13 if_icmplt 5

16 return
```

在 Java 虚拟机中，缺乏对 byte、 char 和 short 类型数据直接操作的支持所带来的问题并不大，因为这些类型的值都在编译过程中就自动被转换为 int 类型（ byte 和 short 带符号扩展为 int 类型， char 零位扩展为 int 类型）。因此对于 byte、 char 和 short 类型的数据均可以用 int 的指令操作。唯一额外的代价是将它们长度扩展至 4 字节。
Java 虚拟机对于 long 和浮点类型（ float 和 double）提供了中等程度的支持， 比起 int类型数据所支持的操作，它们仅缺少了条件转移指令部分，其他操作都与 int 类型具有相同程度的支持。 















