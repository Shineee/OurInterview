# 访问运行时常量池 

很多数值常量， 以及对象、 字段和方法， 都是通过当前类的运行时常量池进行访问。对象的访问将在稍后的§3.8 中讨论。类型为 int、 long、 float 和 double 的数据，以及表示 String实例的引用类型数据的访问将由 ldc、ldc_w 和 ldc2_w 指令实现。
ldc 和 ldc_w 指令用于访问运行时常量池中的对象，包括 String 实例，但不包括 double和 long 类型的值。当使用的运行时常量池的项的数目过多时（多于 256 个， 1 个字节能表示的范围）， 需要使用 ldc_w 指令取代 ldc 指令来访问常量池。ldc2_w 指令用于访问类型为 double和 long 的运行时常量池项，这条指令没有非宽索引的版本（即没有 ldc2 指令）。
对于整型常量，包括 byte、 char、 short 和 int， 如前面§3.2 节所描述，将编译到代码之中， 使用 bipush、 sipush 和 iconst_<i>指令进行访问。某些浮点常量也可以编译进代码，使用 fconst_<f>和 dconst_<d>指令进行访问。
通过以上描述，编译的规则已经基本解释清楚了。 下面这个例子将这些规则汇总了起来：

```java
void useManyNumeric() {

int i = 100;

int j = 1000000;

long l1 = 1;

long l2 = 0xffffffff;

double d = 2.2;

...do some calculations...

}
```

编译后代码如下：

```java
Method void useManyNumeric()

0 bipush 100 // Push a small int with bipush

2 istore_1

3 ldc #1 // Push int constant 1000000; a larger int

// value uses ldc

5 istore_2

6 lconst1 // A tiny long value uses short, fast lconst1

7 lstore_3

8 ldc2_w #6 // Push long 0xffffffff (that is, an int -1); any

// long constant value can be pushed using ldc2_w

11 lstore 5

13 ldc2_w #8 // Push double constant 2.200000; uncommon

// double values are also pushed using ldc2_w

16 dstore 7

...do those calculations... 
```







