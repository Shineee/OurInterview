# Java Exception类



所有的异常类是从java.lang.Exception类继承的子类。

Exception类是Throwable类的子类。除了Exception类外，Throwable还有一个子类Error 。

Java程序通常不捕获错误。错误一般发生在严重故障时，它们在Java程序处理的范畴之外。

Error用来指示运行时环境发生的错误。

例如，JVM内存溢出。一般地，程序不会从错误中恢复。

异常类有两个主要的子类：IOException类和RuntimeException类。

![](img/exception.jpg)



在Java 内置类中(接下来会说明)，有大部分常用检查性和非检查性异常。

## Java 内置异常类

Java 语言定义了一些异常类在java.lang标准包中。

标准运行时异常类的子类是最常见的异常类。由于java.lang包是默认加载到所有的Java程序的，所以大部分从运行时异常类继承而来的异常都可以直接使用。

Java根据各个类库也定义了一些其他的异常，下面的表中列出了Java的非检查性异常。

| **异常**                          | **描述**                                   |
| ------------------------------- | ---------------------------------------- |
| ArithmeticException             | 当出现异常的运算条件时，抛出此异常。例如，一个整数"除以零"时，抛出此类的一个实例。 |
| ArrayIndexOutOfBoundsException  | 用非法索引访问数组时抛出的异常。如果索引为负或大于等于数组大小，则该索引为非法索引。 |
| ArrayStoreException             | 试图将错误类型的对象存储到一个对象数组时抛出的异常。               |
| ClassCastException              | 当试图将对象强制转换为不是实例的子类时，抛出该异常。               |
| IllegalArgumentException        | 抛出的异常表明向方法传递了一个不合法或不正确的参数。               |
| IllegalMonitorStateException    | 抛出的异常表明某一线程已经试图等待对象的监视器，或者试图通知其他正在等待对象的监视器而本身没有指定监视器的线程。 |
| IllegalStateException           | 在非法或不适当的时间调用方法时产生的信号。换句话说，即 Java 环境或 Java 应用程序没有处于请求操作所要求的适当状态下。 |
| IllegalThreadStateException     | 线程没有处于请求操作所要求的适当状态时抛出的异常。                |
| IndexOutOfBoundsException       | 指示某排序索引（例如对数组、字符串或向量的排序）超出范围时抛出。         |
| NegativeArraySizeException      | 如果应用程序试图创建大小为负的数组，则抛出该异常。                |
| NullPointerException            | 当应用程序试图在需要对象的地方使用 `null` 时，抛出该异常         |
| NumberFormatException           | 当应用程序试图将字符串转换成一种数值类型，但该字符串不能转换为适当格式时，抛出该异常。 |
| SecurityException               | 由安全管理器抛出的异常，指示存在安全侵犯。                    |
| StringIndexOutOfBoundsException | 此异常由 `String` 方法抛出，指示索引或者为负，或者超出字符串的大小。  |
| UnsupportedOperationException   | 当不支持请求的操作时，抛出该异常。                        |

下面的表中列出了Java定义在java.lang包中的检查性异常类。

| **异常**                     | **描述**                                   |
| -------------------------- | ---------------------------------------- |
| ClassNotFoundException     | 应用程序试图加载类时，找不到相应的类，抛出该异常。                |
| CloneNotSupportedException | 当调用 `Object` 类中的 `clone` 方法克隆对象，但该对象的类无法实现 `Cloneable` 接口时，抛出该异常。 |
| IllegalAccessException     | 拒绝访问一个类的时候，抛出该异常。                        |
| InstantiationException     | 当试图使用 `Class` 类中的 `newInstance` 方法创建一个类的实例，而指定的类对象因为是一个接口或是一个抽象类而无法实例化时，抛出该异常。 |
| InterruptedException       | 一个线程被另一个线程中断，抛出该异常。                      |
| NoSuchFieldException       | 请求的变量不存在                                 |
| NoSuchMethodException      | 请求的方法不存在                                 |

## 异常方法

下面的列表是Throwable 类的主要方法:

| **序号** | **方法及说明**                                |
| ------ | ---------------------------------------- |
| 1      | **public String getMessage()**返回关于发生的异常的详细信息。这个消息在Throwable 类的构造函数中初始化了。 |
| 2      | **public Throwable getCause()**返回一个Throwable 对象代表异常原因。 |
| 3      | **public String toString()**使用getMessage()的结果返回类的串级名字。 |
| 4      | **public void printStackTrace()**打印toString()结果和栈层次到System.err，即错误输出流。 |
| 5      | **public StackTraceElement [] getStackTrace()**返回一个包含堆栈层次的数组。下标为0的元素代表栈顶，最后一个元素代表方法调用堆栈的栈底。 |
| 6      | **public Throwable fillInStackTrace()**用当前的调用栈层次填充Throwable 对象栈层次，添加到栈层次任何先前信息中。 |