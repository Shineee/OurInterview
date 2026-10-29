# 抛出异常和处理异常 

程序中使用 throw 关键字来抛出异常，它的编译过程很简单，譬如：

```java
void cantBeZero(int i) throws TestExc {

if (i == 0) {

throw new TestExc();

}

}
```

编译后代码如下：

```java
Method void cantBeZero(int)

0 iload_1 // Push argument 1 (i)

1 ifne 12 // If i==0, allocate instance and throw

4 new #1 // Create instance of TestExc

7 dup // One reference goes to the constructor

8 invokespecial #7 // Method TestExc.<init>()V

11 athrow // Second reference is thrown

12 return // Never get here if we threw TestExc

try-catch 结构的编译也同样简单，譬如：

void catchOne() {

try {

tryItOut();

} catch (TestExc e) {

handleExc(e);

} 

}
```

编译后代码如下：

```java
Method void catchOne()

0 aload_0 // Beginning of try block

1 invokevirtual #6 // Method Example.tryItOut()V

4 return // End of try block; normal return

5 astore_1 // Store thrown value in local variable 1

6 aload_0 // Push this

7 aload_1 // Push thrown value

8 invokevirtual #5 // Invoke handler method:

// Example.handleExc(LTestExc;)V

11 return // Return after handling TestExc

Exception table:

From To Target Type

0 4 5 Class TestExc
```

仔细看可发现， try 语句块被编译后似乎没有生成任何指令，就像它没有出现过一样。

```java
Method void catchOne()

0 aload_0 // Beginning of try block

1 invokevirtual #4 // Method Example.tryItOut()V

4 return // End of try block; normal return
```

如果在 try 语句块执行过程中没有异常抛出， 程序那么就犹如没有使用 try 结构一样：在tryItOut()调用 catchOne()方法后就返回了。
在 try 语句块之后， Java 虚拟机代码实现的一个 catch 语句如下：

```
6 aload_0 // Push this

7 aload_1 // Push thrown value

8 invokevirtual #5 // Invoke handler method:

// Example.handleExc(LTestExc;)V

11 return // Return after handling TestExc

Exception table:

From To Target Type

0 4 5 Class TestExc
```

在 catch 语句块里， 调用 handleExc()方法的指令和正常的方法调用完全一样。不过，每个 catch 语句块的会使编译器在异常表中增加一个成员（即一个异常处理器， §2.10, §4.7.3）。 catchOne()方法的异常表中有一个成员，这个成员对应 catchOne()方法 catch 语句块的一个可捕获的异常参数（本例中为 TestExc 的实例）。在 catchOne()的执行过程中，如果在它的编译代码第 0 至 4 句之间有 TestExc 异常实例被抛出，那么操作将转移至第 5 句继续执 行，即进入 catch 语句块的实现步骤。如果抛出的异常不是 TestExc 实例，那么 catchOne()的 catch 语句块则不能捕获它，这个异常将被抛出给 catchOne()方法的调用者。

一个 try 结构中可包含多个 catch 语句块，譬如：

```java
void catchTwo() {

try {

tryItOut();

} catch (TestExc1 e) {

handleExc(e);

} catch (TestExc2 e) {

handleExc(e);

}

}
```

如果 try 语句包含有多个 catch 语句块，那么在编译代码中，多个 catch 语句块的内容将连续排列，在异常表中也会有对应的连续排列的成员，它们的排列的顺序和源码中的 catch 语句块出现的顺序一致。

```java
Method void catchTwo()

0 aload_0 // Begin try block

1 invokevirtual #5 // Method Example.tryItOut()V

4 return // End of try block; normal return

5 astore_1 // Beginning of handler for TestExc1;

// Store thrown value in local variable 1

6 aload_0 // Push this

7 aload_1 // Push thrown value

8 invokevirtual #7 // Invoke handler method:

// Example.handleExc(LTestExc1;)V

11 return // Return after handling TestExc1

12 astore_1 // Beginning of handler for TestExc2;

// Store thrown value in local variable 1

13 aload_0 // Push this

14 aload_1 // Push thrown value

15 invokevirtual #7 // Invoke handler method:

// Example.handleExc(LTestExc2;)V

18 return // Return after handling TestExc2

Exception table:

From To Target Type

0 4 5 Class TestExc1

0 4 12 Class TestExc2
```

catchTwo()在执行时，如果 try 语句块中（ 编译代码的第 1 至 4 句） 抛出了一个异常，这个异常可以被多个 catch 语句块捕获（ 即这个异常的实例是一个或多个 catch 语句块的参数）， 则 Java 虚拟机将选择第一个（ 最上层）catch 语句块来处理这个异常。 程序将转移至这个 catch语句块对应的 Java 虚拟机代码块中继续执行。如果抛出的异常不是任何 catch 语句块的参数 （ 即不能被捕获），那么 Java 虚拟机将把这个异常抛出给 catchTwo()方法的调用者， catchTwo()方法本身的所有 catch 语句块的编译代码都不会被执行。

在 try-catch 语句可以嵌套使用，编译后产生的编译代码和一个 try 语句对应多个 catch语句的结构很相似，譬如：

```java
void nestedCatch() {

try {

try {

tryItOut();

} catch (TestExc1 e) {

handleExc1(e);

}

} catch (TestExc2 e) {

handleExc2(e);

}

}
```

编译后代码如下：

```java
Method void nestedCatch()

0 aload_0 // Begin try block

1 invokevirtual #8 // Method Example.tryItOut()V

4 return // End of try block; normal return

5 astore_1 // Beginning of handler for TestExc1;

// Store thrown value in local variable 1

6 aload_0 // Push this

7 aload_1 // Push thrown value

8 invokevirtual #7 // Invoke handler method:

// Example.handleExc1(LTestExc1;)V

11 return // Return after handling TestExc1

12 astore_1 // Beginning of handler for TestExc2;

// Store thrown value in local variable 1

13 aload_0 // Push this

14 aload_1 // Push thrown value

15 invokevirtual #6 // Invoke handler method:

// Example.handleExc2(LTestExc2;)V

18 return // Return after handling TestExc2

Exception table:

From To Target Type

0 4 5 Class TestExc1 

0 12 12 Class TestExc2

```

try-catch 语句的嵌套关系只体现在异常表之中， Java 虚拟机本身并不要求异常表中成员（ §2.10）的顺序，但是编译器需要保证 try-catch 语句是有结构顺序的，编译器会根据 catch语句在代码中的顺序对异常处理表进行排序， 以保证在代码任何位置抛出的任何异常， 都会被最接近异常抛出位置的、可处理该异常的 catch 语句块所处理。
例如， 如果在编译代码第 1 句， 即 tryItOut()方法执行过程中抛出一个 TestExc1 异常的实例，这个异常实例将被调用 handleExc1()方法的 catch 语句块处理。 即使这个异常是发生在外层 catch 语句（捕获 TestExc2 异常的 catch 语句）的处理范围之内，并且外层 catch 语句也同样声明了能处理这类异常，但依然不会分配给外部的 catch 语句来处理，因为它在异常处理表中顺序在内部异常之后。
还有一个微妙之处需要注意， catch 语句块的处理范围包括 from 但不包括 to 所表示的偏移量本身（ §4.7.3）。即 catch 语句块中， TestExc1 在异常表所对应的异常处理器并没有覆盖到字节偏移量为 4 处的返回指令。不过， TestExc2 在异常表中对应的异常处理器却覆盖了偏移量为 11 处的返回指令。 因此在此场景下，如果在嵌套 catch 语句块中如果返回指令抛出了异常，将由外层的异常处理器进行处理。 