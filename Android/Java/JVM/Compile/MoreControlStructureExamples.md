# 更多的控制结构示例 

在§3.2 小节中展示了一些控制结构是如何编译的。在 Java 语言中还有很多其他的控制结构（ if-then-else、 do、 while、 break 以及 continue） 也有特定的编译规则。本规范将在“编译 switch 语句块”、 “抛出异常和处理异常” 和“编译 finally 语句块”中分别讨论关于 switch 语句块，异常和 finally 语句块的编译规则。
下面这个例子是关于 while 的循环的编译规则， Java 虚拟机会根据数据的变化而生成不同的条件跳转语句， 通常情况下， Java 虚拟机对 int 类型数据提供的支持最为完善。

```java
void whileInt() {

int i = 0;

while (i < 100) {

i++;

}

}
```

编译后代码如下：

```java
Method void whileInt()

0 iconst_0

1 istore_1

2 goto 8

5 iinc 1 1

8 iload_1

9 bipush 100

11 if_icmplt 5

14 return
```

注意到 while 语句的条件判断（由 if_icmplt 指令实现） 在 Java 虚拟机编译代码中的循环的最底部，这和 spin()方法的条件判断位置一致。 原本处于循环最底部、在迭代结束时执行的条件测试指令被一条 goto 指令强制跳转到在循环的第一次迭代之前执行。这样如果条件判断失败，则不会在进入循环体， 其他额外的指令就不会执行了。 不过 while 循环通常都使用在期望循环体会被执行的场景之中（而不是用来当作 if 语句使用）。在接下来的循环迭代中， 由于条件判断的操作被置于循环体的底部，这样相当于在执行循环体时节省了一条 Java 虚拟机指令。如果将条件判断的操作放在循环体的顶部，那循环体就必须在尾部额外增加一条 goto 指令用于循环体结束时跳转回顶部。
虚拟机对各种数据类型的控制结构采用了相似的方式编译，只是根据不同数据类型使用不同的指令来访问。这么做多少会使编译代码效率降低，因为这样可能需要更多的 Java 虚拟机指令来实现，譬如： 

```java
void whileDouble() {

double i = 0.0;

while (i < 100.1) {

i++;

}

}
```

编译后代码如下：

```java
Method void whileDouble()

0 dconst_0

1 dstore_1

2 goto 9

5 dload_1

6 dconst_1

7 dadd

8 dstore_1

9 dload_1

10 ldc2_w #4 // Push double constant 100.1

13 dcmpg // To do the compare and branch we have to use...

14 iflt 5 // ...two instructions

17 return
```

每个浮点型数据都有两条比较指令：对于 float 类型是 fcmpl 和 fcmpg 指令，对于 double是 dcmpl 和 dcmpg 指令。这些指令语义相似， 仅仅在对待 NaN 变量时有所区别。 NaN 是无序的，所以如果有其中一个操作数为NaN， 则所有浮点型的比较指令都失败①。无论是比较操作是否会因为遇到 NaN 值而失败，编译器都会根据不同的操作数类型来选择不同的比较指令，譬如：

```java
int lessThan100(double d) {

if (d < 100.0) {

return 1;

} else {

return -1;

}

}
```

编译后代码如下： 

```java
Method int lessThan100(double)

0 dload_1

1 ldc2_w #4 // Push double constant 100.0

4 dcmpg // Push 1 if d is NaN or d > 100.0;

// push 0 if d == 100.0

5 ifge 10 // Branch on 0 or 1

8 iconst_1

9 ireturn

10 iconst_m1

11 ireturn
```

如果 d 不是 NaN 并且小于 100.0， dcmpg 指令将 int 值-1 压入操作数栈， ifge 指令就不会被执行。如果 d 大于 100.0 或者是 NaN， dcmpg 指令将 int 值 1 压入操作数栈， ifge 指令将执行。如果 d 等于 100.0， dcmpg 指令将 int 值 0 压入操作数栈， ifge 指令将执行。

如果比较逻辑相反， dcmpl 指令可以达到相同的效果， 譬如：

```java
int greaterThan100(double d) {

if (d > 100.0) {

return 1;

} else {

return -1;

}

}
```

编译后代码如下：

```java
Method int greaterThan100(double)

0 dload_1

1 ldc2_w #4 // Push double constant 100.0

4 dcmpl // Push -1 if d is Nan or d < 100.0;

// push 0 if d == 100.0

5 ifle 10 // Branch on 0 or -1

8 iconst_1

9 ireturn

10 iconst_m1

11 ireturn
```

再来分析一次，无论是否因参数 d 传入了 NaN 值而导致的比较失败， dcmpl 指令都会向操作数栈压入一个 int 型值，使程序进入 ifle 指令的分支。如果没有 dcmpl 和 dcmpg 指令，那么对于实例中的方法，编译器不得不因 NaN 值而做更多处理。 



























