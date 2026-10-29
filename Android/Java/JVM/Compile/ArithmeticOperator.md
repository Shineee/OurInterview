# 算术运算符 

Java 虚拟机通常基于操作数栈来进行算术运算（只有 iinc 指令例外，它直接对局部变量进行自增操作）， 譬如下面的 align2grain()方法， 它的作用是将 int 值对齐到 2 的指定幂次： 

```java
int align2grain(int i, int grain) {

return ((i + grain-1) & ~(grain-1));

}
```

算术运算使用到的操作数都是从操作数栈中弹出的， 运算结果被压回操作数栈中。在内部运算时， 中间运算（ Arithmetic Subcomputations） 的结果可以被当作操作数使用。 譬如~(grain-1)的值就是被这样使用的：

```java
5 iload_2 // Push grain

6 iconst_1 // Push int constant 1

7 isub // Subtract; push result

8 iconst_m1 // Push int constant -1

9 ixor // Do XOR; push result
```

首先， grain-1 的结果由第 2 个局部变量和立即操作数 int 数值 1 的计算得出。参与运算的操作数从操作数栈中弹出，然后它们将被改变，最后再入栈到操作数栈之中。这里被改变是单个操作数的算术指令 ixor 运算的结果（因为~x == -1^x）。类似地， ixor 指令的结果是接下来将作为 iand 指令的操作数被使用。
整个方法的编译代码如下：

```java
Method int align2grain(int,int)

0 iload_1

1 iload_2

2 iadd

3 iconst_1

4 isub

5 iload_2

6 iconst_1

7 isub

8 iconst_m1

9 ixor

10 iand

11 ireturn
```















