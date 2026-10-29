# 编译 switch 语句 

编译器会使用 tableswitch 和 lookupswitch 指令来生成 switch 语句的编译代码。tableswitch 指令用于表示 switch 结构中的 case 语句块，可以高效地从索引表中确定 case语句块的分支偏移量。当 switch 语句中提供的条件值不能从索引表中确定任何一个 case 语句块的分支偏移量时， default 分支将起作用。譬如：

```java
int chooseNear(int i) {

switch (i) {

case 0: return 0;

case 1: return 1;

case 2: return 2;

default: return -1;

}

}
```

编译后代码如下： 

```java
Method int chooseNear(int)

0 iload_1 // Push local variable 1 (argument i)

1 tableswitch 0 to 2: // Valid indices are 0 through 2

0: 28 // If i is 0, continue at 28

1: 30 // If i is 1, continue at 30

2: 32 // If i is 2, continue at 32

default:34 // Otherwise, continue at 34

28 iconst_0 // i was 0; push int constant 0...

29 ireturn // ...and return it

30 iconst_1 // i was 1; push int constant 1...

31 ireturn // ...and return it

32 iconst_2 // i was 2; push int constant 2...

33 ireturn // ...and return it

34 iconst_m1 // otherwise push int constant –1...

35 ireturn // ...and return it
```

Java 虚拟机的 tableswitch 和 lookupswitch 指令都只能支持 int 类型的条件值。 选择支持 int 类型是因为 byte、char 和 short 类型的值都会被隐式展为 int 型。如果 chooseNear()方法中使用 short 类型作为条件值， 那编译出来的代码中与使用 int 类型时是完全相同的。 如果使用其他数值类型的条件值，那就必须窄化转换成 int 类型。
当 switch 语句中的 case 分支的条件值比较稀疏时，tableswitch 指令的空间使用率偏低。这种情况下将使用 lookupswitch 指令来替代。 lookupswitch 指令的索引表由 int 型的键值（来源于 case 语句块后面的数值）与对应的目标语句偏移量所构成。当 lookupswitch 指令执行时， switch 语句的条件值将和索引表中的 key 进行比较，如果某个 key 和条件值相符，那么将转移到这个 key 对应的分支偏移量继续继续执行，如果没有 key 值符合，执行将在 default分支执行。譬如：

```java
int chooseFar(int i) {

switch (i) {

case -100: return -1;

case 0: return 0;

case 100: return 1;

default: return -1;

}

}
```

编译后的代码如下，相比 chooseNear()方法的编译代码，仅仅把 tableswitch 指令换成了 lookupswitch 指令：

```java
Method int chooseFar(int) 

0 iload_1

1 lookupswitch 3:

-100: 36

0: 38

100: 40

default:42

36 iconst_m1

37 ireturn

38 iconst_0

39 ireturn

40 iconst_1

41 ireturn

42 iconst_m1

43 ireturn
```

Java 虚拟机规定的 lookupswitch 指令的索引表必须根据 key 值排序，这样使用（如采用二分搜索）将会比直接使用线性扫描搜索来得更有效率。在从索引表确定分支偏移量的过程中，lookupswitch 指令是把条件值与不同的 key 的进行比较， 而 tableswitch 指令则只需要索引值进行一次范围检查。因此，在如果不需要考虑空间效率时， tableswitch 指令相比lookupswitch 指令有更高的执行效率。 































