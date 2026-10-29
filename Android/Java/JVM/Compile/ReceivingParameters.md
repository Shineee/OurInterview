# 接收参数 

如果传递了 n 个参数给某个实例方法，则当前栈帧会按照约定的顺序接收这些参数，将它们保存为方法的第 1 个至第 n 个局部变量之中。譬如：

```java
int addTwo(int i, int j) {

return i + j;

}
```

编译后代码如下：

```java
Method int addTwo(int,int)

0 iload_1 // Push value of local variable 1 (i)

1 iload_2 // Push value of local variable 2 (j)

2 iadd // Add; leave int result on operand stack

3 ireturn // Return int result
```

按照约定，实例方法需要传递一个自身实例的引用作为第 0 个局部变量。在 Java 语言中自身实例可以通过 this 关键字来访问。
类（ static） 方法不需要传递实例引用，所以它们不需要使用第 0 个局部变量来保存 this关键字。如果 addTwo()是类方法，那么接收的参数和之前示例相比略有不同：

```java
static int addTwoStatic(int i, int j) {

return i + j;

}
```

编译后代码如下：

```java
Method int addTwoStatic(int,int)

0 iload_0

1 iload_1

2 iadd

3 ireturn
```

两段代码唯一的区别是， 后者方法保存参数到局部变量表时，是从编号为 0 的局部变量开始而不是 1。 



























