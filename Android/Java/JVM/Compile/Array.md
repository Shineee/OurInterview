# 数组 

在 Java 虚拟机中，数组也使用对象来表示。数组由专门的指令集来创建和操作。 newarray指令用于创建元素类型为数值类型的数组。譬如：

```java
void createBuffer() {

int buffer[];

int bufsz = 100;

int value = 12;

buffer = new int[bufsz];

buffer[10] = value;

value = buffer[11];

}
```

编译后代码如下：

```java
Method void createBuffer()

0 bipush 100 // Push int constant 100 (bufsz)

2 istore_2 // Store bufsz in local variable 2

3 bipush 12 // Push int constant 12 (value)

5 istore_3 // Store value in local variable 3 

6 iload_2 // Push bufsz...

7 newarray int // ...and create new array of int of that length

9 astore_1 // Store new array in buffer

10 aload_1 // Push buffer

11 bipush 10 // Push int constant 10

13 iload_3 // Push value

14 iastore // Store value at buffer[10]

15 aload_1 // Push buffer

16 bipush 11 // Push int constant 11

18 iaload // Push value at buffer[11]...

19 istore_3 // ...and store it in value

20 return

```

anewarray 指令用于创建元素为引用类型的一维数组。譬如：

```java
void createThreadArray() {

Thread threads[];

int count = 10;

threads = new Thread[count];

threads[0] = new Thread();

}
```

编译后代码如下：

```java
Method void createThreadArray()

0 bipush 10 // Push int constant 10

2 istore_2 // Initialize count to that

3 iload_2 // Push count, used by anewarray

4 anewarray class #1 // Create new array of class Thread

7 astore_1 // Store new array in threads

8 aload_1 // Push value of threads

9 iconst_0 // Push int constant 0

10 new #1 // Create instance of class Thread

13 dup // Make duplicate reference...

14 invokespecial #5 // ...to pass to instance initialization method

// Method java.lang.Thread.<init>()V

17 aastore // Store new Thread in array at 0

18 return
```

anewarray 指令也可以用于创建多维数组的第一维。 不过我们也可以选择采用multianewarray 指令一次性创建多维数组。 譬如三维数组：

```java
int[] create3DArray() {

int grid[];

grid = new int10[]; 

return grid;

}
```

编译后代码如下：

```java
Method int create3DArray()[]

0 bipush 10 // Push int 10 (dimension one)

2 iconst_5 // Push int 5 (dimension two)

3 multianewarray #1 dim #2 // Class [[[I, a three

// dimensional int array;

// only create first two

// dimensions

7 astore_1 // Store new array...

8 aload_1 // ...then prepare to return it

9 areturn
```

multianewarray 指令的第一个操作数是运行时常量池索引，它表示将要被创建的数组的成员类型。第二个操作数是需要创建的数组的实际维数。 multianewarray 指令可以用于创建所有类型的多维数组， 譬如 create3DArray 中展示的。注意，多维数组也只是一个对象，所以使用aload_1 指令加载， 使用 areturn 指令返回。
所有的数组都有一个与之关联的长度属性，通过 arraylength 指令访问。 





























