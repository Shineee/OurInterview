# 编译 finally 语句块 

前面提到过，本章所使用的编译代码由 Oracle 的 1.0.2 版本 JDK 的 javac 编译器生成，因此这里的 Class 文件版本必定低于 50.0， 即将会使用 jsr 指令来编译 finally 语句块，更多内容请参见 4.10.2.5 节的“异常与 finally” ①。
编译 try-finally 语句和编译 try-catch 语句基本相同。在代码执行完 try 语句之前（无论有没有抛出异常）， finally 语句块中的内容都会被执行，譬如：

```java
void tryFinally() {

try {

tryItOut();

} finally {

wrapItUp();

}

}
```

编译后代码如下：

```java
Method void tryFinally()

0 aload_0 // Beginning of try block

1 invokevirtual #6 // Method Example.tryItOut()V

4 jsr 14 // Call finally block

7 return // End of try block

8 astore_1 // Beginning of handler for any throw

9 jsr 14 // Call finally block

12 aload_1 // Push thrown value

13 athrow // ...and rethrow the value to the invoker

14 astore_2 // Beginning of finally block

15 aload_0 // Push this

16 invokevirtual #5 // Method Example.wrapItUp()V

19 ret 2 // Return from finally block

Exception table:

From To Target Type

0 4 8 any
```

有四种方式可以让程序退出 try 语句： 1.语句块所有正常执行结束； 2.通过 return 语句退出方法； 3.通过 break 或 continue 语句退出循环； 4.抛出异常。如果 tryItOut()正常结束（ 没有抛出异常） 并返回， 后面的 jsr 指令会使程序跳转到 finally 语句块继续执行。编译代码中的第 4 句的“ jsr 14”表示的意思是“调用程序子片段（ Subroutine Call）”，这条指令使程序跳转至第 14 句的 finally 语句块（ finally 语句块的内容被编译为一段程序子片段）的实现代码之中。当 finally 语句块运行结束，使用“ ret 2”指令将程序返回至 jsr 指令（ 即第 4
句） 的下一句继续执行。
调用程序子片段的更多细节如下： 在本例中， jsr 指令将其下一条指令的地址（ 即第 7 句的return 指令） 在程序跳转前压入操作数栈。 程序跳转后使用 astore_2 指令将栈顶的元素（ 即return 指令的地址） 保存在第 2 个局部变量中。然后，执行 finally 语句块（ 在这个例子中，finally 语句块的内容包括 aload_0 和 invokevirtual 两条指令）。当 finally 语句块的代码正常地执行结束后， ret 指令使程序跳转至第 2 个局部变量所保存的地址（ 即 return 指令的地址）继续执行，至此 tryFinally()运行结束。
一个带有 finally 语句块的 try 语句， 在编译时会生成一个特殊的异常处理器，这个异常处 理器可以捕获 try 语句中抛出的所有异常。当 tryItOut()抛出异常时， Java 虚拟机会在tryFinally()方法则在异常处理器表中寻找一个合适的异常处理器。这个处理器被找到后，会转到异常处理器实现代码处（ 这个例子中为编译代码的第 8 句） 继续执行。编译代码第 8 句的astore_1 指令用于将抛出的异常保存在第 1 个局部变量中。接下来的 jsr 指令调用 finally语句块的程序子片段。如果正常返回（ finally 语句块正常运行结束）， 位于编译代码第 12 句的aload_1 指令将抛出的异常压入操作数栈顶， 再接下来的 athrow 指令将异常抛出给tryFinally()方法的调用者。

一个 try 语句中同时带有 catch 和 finally 语句块的编译示例如下：

```java
void tryCatchFinally() {

try {

tryItOut();

} catch (TestExc e) {

handleExc(e);

} finally {

wrapItUp();

}

}
```

编译后代码如下：

```java
Method void tryCatchFinally()

0 aload_0 // Beginning of try block

1 invokevirtual #4 // Method Example.tryItOut()V

4 goto 16 // Jump to finally block

7 astore_3 // Beginning of handler for TestExc;

// Store thrown value in local variable 3

8 aload_0 // Push this

9 aload_3 // Push thrown value

10 invokevirtual #6 // Invoke handler method:

// Example.handleExc(LTestExc;)V

13 goto 16 // Huh???

16 jsr 26 // Call finally block

19 return // Return after handling TestExc

20 astore_1 // Beginning of handler for exceptions

// other than TestExc, or exceptions

// thrown while handling TestExc

21 jsr 26 // Call finally block

24 aload_1 // Push thrown value...

25 athrow // ...and rethrow the value to the invoker

26 astore_2 // Beginning of finally block 



27 aload_0 // Push this

28 invokevirtual #5 // Method Example.wrapItUp()V

31 ret 2 // Return from finally block

Exception table:

From To Target Type

0 4 7 Class TestExc

0 16 20 any
```

严格来说，上面代码中的 goto 指令是没有必要，但是 Oracle 的 1.0.2 版本的 JDK 的 javac编译器会产生这条指令。
如果 try 语句块中所有指令都正常执行结束，第 4 句的 goto 指令将程序跳转至第 16 句的finally 语句块之中。在第 26 句， finally 语句块执行结束， 程序将跳转回第 19 句的 return指令，至此 tryCatchFinally()方法运行结束。
如果 tryItOut()方法中抛出了并非 TestExc 异常的异常实例，又或者 catch 语句块中的handleExc()抛出异常，则这时候异常表中对应为第二个异常处理器就会生效。此时程序将跳转至第 20 句，进入这个异常处理器的代码， 将前面抛出的异常保存为第 1 个局部变量中，第 26 句的 finally 语句块一样会被调用。 在 tryCatchFinally()方法结束时，这个异常会从第 1 个局部变量中取出， 并通过 athrow 指令抛给方法调用者。如果在 finally 子句中有任何的异常抛出，则 finally 语句块停止运行， tryCatchFinally()方法异常退出，并抛出新出现的异常给tryCatchFinally()方法的调用者。 













