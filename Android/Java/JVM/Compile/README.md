# JVM 编译 

Java 虚拟机是为了支持 Java 语言而的设计的。 Oracle 的 JDK 包括两部分内容：一部分是将 Java 源代码编译成 Java 虚拟机的指令集的编译器，另一部分是用于 Java 虚拟机的运行时环境。理解编译器是如何与 Java 虚拟机协同工作的，对编译器开发人员来说很有好处，同样也有助于理解 Java 虚拟机本身。
请注意：术语“编译器”在某些上下文场景中专指把 Java 虚拟机的指令集转换为特定 CPU指令集的翻译器。譬如即时代码生成器（ Just-In-Time/JIT Code Generator）就是一种在Class 文件中的代码被 Java 虚拟机代码加载后，生成与平台相关的特定指令的编译器。但是在本章中讨论的编译器将不考虑这类代码生成的问题，只会涉及到从使用 Java 语言编写的源代码编译为 Java 虚拟机指令集的编译器。

### [示例的格式说明](SampleFormatDescription.md)

### [常量、 局部变量的使用和控制结构](TheUseAndControlOfConstantAndLocalVariables.md)

### [算术运算符](ArithmeticOperator.md)

### [访问运行时常量池](AccessRuntimeConstantPool.md)

### [更多的控制结构示例](MoreControlStructureExamples.md)

### [接收参数](ReceivingParameters.md)

### [方法调用](MethodCall.md)

### [使用类实例](UseClassInstance.md)

### [数组](Array.md)

### [编译 switch 语句](CompilerSwitchStatement.md)

### [使用操作数栈](OperandStack.md)

### [抛出异常和处理异常](ThrowExceptionAndExceptionHandling.md)

### [编译 finally 语句块](CompilerFinallyStatementBlock.md)

### [同步](Synchronization.md)

### [注解](Annotation.md)