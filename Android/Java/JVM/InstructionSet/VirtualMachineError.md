# 虚拟机错误 

当 Java 虚拟机出现了内部错误，或者由于资源限制导致虚拟机无法实现 Java 语言中的语义时， Java 虚拟机将会抛出一个属于 VirtualMachineError 的子类的异常对象实例。本规范无法预测虚拟机会遇到哪些内部错误或者资源受限的情况，也不可能精确地指出虚拟机何时会发生这些异常情况。因此，下面定义的这些 VirtualMachineError 的子类异常可能会出现在 Java 虚拟机运作过程中的任意时刻：

* InternalError： Java 虚拟机实现的软件或硬件错误都会导致 InternalError 异常的出现， InternalError 是一个典型的异步异常（ §2.10），它可能出现在程序中的任何位置。
* OutOfMemoryError：当 Java 虚拟机实现耗尽了所有虚拟和物理内存，并且内存自动管理子系统无法回收到足够共新对象分配所需的内存空间时，虚拟机将抛出OutOfMemoryError 异常。
* StackOverflowError： 当 Java 虚拟机实现耗尽了线程全部的栈空间，这种情况经常是由于程序执行时无限制的递归调用而导致的，虚拟机将会抛出 StackOverflowError异常。
* UnknownError：当某种异常或错误出现，但虚拟机实现无法确定具体实际是哪种异常或错误的时候，将会抛出 UnknownError 异常。 