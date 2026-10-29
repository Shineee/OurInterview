# 同步 

Java 虚拟机中的同步（ Synchronization）基于进入和退出管程（ Monitor） 对象实现。无论是显式同步（有明确的 monitorenter 和 monitorexit 指令）还是隐式同步（依赖方法调用和返回指令实现的）都是如此。
在 Java 语言中， 同步用的最多的地方可能是被 synchronized 修饰的同步方法。 同步方法并不是由monitorenter 和 monitorexit 指令来实现同步的， 而是由方法调用指令读取运行时常量池中方法的 ACC_SYNCHRONIZED 标志来隐式实现的（ 参见§2.11.10 “同步”）。
monitorenter 和 monitorexit 指令用于实现同步语句块，譬如：

```java
void onlyMe(Foo f) {

synchronized(f) { 

doSomething();

}

}
```

编译后代码如下：

```java
Method void onlyMe(Foo)

0 aload_1 // Push f

1 dup // Duplicate it on the stack

2 astore_2 // Store duplicate in local variable 2

3 monitorenter // Enter the monitor associated with f

4 aload_0 // Holding the monitor, pass this and...

5 invokevirtual #5 // ...call Example.doSomething()V

8 aload_2 // Push local variable 2 (f)

9 monitorexit // Exit the monitor associated with f

10 goto 18 // Complete the method normally

13 astore_3 // In case of any throw, end up here

14 aload_2 // Push local variable 2 (f)

15 monitorexit // Be sure to exit the monitor!

16 aload_3 // Push thrown exception...

17 athrow // ...then rethrow the value to the invoker

18 return // Return in the normal case

Exception table:

FromTo Target Type

4 10 13 any

13 16 13 any
```

编译器必须确保无论方法通过何种方式完成， 方法中调用过的每条 monitorenter 指令都必须有执行其对应 monitorexit 指令， 而无论这个方法是正常结束（ §2.6.4） 还是异常结束（ §2.6.5）。 为了保证在方法异常完成时 monitorenter 和 monitorexit 指令依然可以正确配对执行，编译器会自动产生一个异常处理器（ §2.10），这个异常处理器声明可处理所有的异常， 它的目的就是用来执行 monitorexit 指令。 





