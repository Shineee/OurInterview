# 使用操作数栈 

Java 虚拟机为方便使用操作数栈，提供了的大量的不区分操作数栈数据类型的指令。这些指令都很常用，因为 Java 虚拟机是基于栈的虚拟机，大量操作是建立在操作数栈的基础之上的。譬如：

```java
public long nextIndex() {

return index++;

}

private long index = 0;
```

编译后代码如下：

```java
Method long nextIndex()

0 aload_0 // Push this

1 dup // Make a copy of it

2 getfield #4 // One of the copies of this is consumed

// pushing long field index, 

// above the original this

5 dup2_x1 // The long on top of the operand stack is

// inserted into the operand stack below the

// original this

6 lconst_1 // Push long constant 1

7 ladd // The index value is incremented...

8 putfield #4 // ...and the result stored back in the field

11 lreturn // The original value of index is left on

// top of the operand stack, ready to be returned

```

注意， Java 虚拟机不允许作用于操作数栈的指令修改或者拆分那些不可拆分操作数（如存放long 或 double 的操作数）。 









