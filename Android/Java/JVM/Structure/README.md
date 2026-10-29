# JVM 结构

本规范描述的是一种抽象化的虚拟机的行为，而不是任何一种（译者注：包括Oracle 公司自己的 HotSpot 和 JRockit 虚拟机）被广泛使用的虚拟机实现。如果只是要去“正确地”实现一台 Java 虚拟机，其实并不如大多数人所想的那样高深和困难——只需要正确读取 Class 文件之中每一条字节码指令，并且能正确执行这些指令所蕴含的操作即可。所有在虚拟机规范之中没有明确描述的实现细节，都不应成为虚拟机设计者发挥创造性的牵绊，设计者可以完全自主决定所有规范中不曾描述的虚拟机内部细节，例如：运行时数据区的内存如何布局、选用哪种垃圾收集的算法、是否要对虚拟机字节码指令进行一些内部优化操作（如使用即时编译器把字节码编译为机器码）。在本规范之中所有关于 Unicode 的描述，都是基于 Unicode 6.0.0 标准，相关资料读者可以在 Unicode 的网站（ http://www.unicode.org）中查找到。

### [Class 文件格式](ClassFileFormat.md)

### [数据类型](DataType.md)

### [原始类型与值](PrimitiveTypeAndValue.md)

### [引用类型与值](ReferenceTypeAndValue.md)

### [运行时数据区](RunTimeDataArea.md)

### [栈帧](StackFrame.md)

### [对象的表示](ObjectRepresentation.md)

### [浮点算法](FloatingPointArithmetic.md)

### [初始化方法的特殊命名 ](SpecialNamingOfInitializationMethod.md)

### [异常](Abnormal.md)

### [字节码指令集简介](ByteCodeInstructionSet.md)

### [类库](ClassLibraries.md)



