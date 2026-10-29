# 示例的格式说明 

本章节的示例主要包括有源文件和 Java 虚拟机代码注解列表（ Annotated Listings），其中， Java 虚拟机的代码注解列表是由 Oracle 的 1.0.2 版本的 JDK 的 javac 编译器生成。Java 虚拟机代码将使用 Oracle 的 javap 工具所生成的非正式的“虚拟机汇编语言（ Virtual Machine Assembly Language）”格式来描述。读者可以自行使用 javap 命令去查看更多已编译方法的例子。
如果读者阅读过汇编代码，应该很熟悉示例中的格式。所有指令的格式如下：

```java
<index> <opcode> <operand1> [<operand2>...]
```

<index>是 code[]数组中的指令的操作码的索引， 此处的 code[]数组就是存储当前方法的Java 虚拟机字节码的 Code 属性中的 code[]数组（ §4.7.3）。也可以认为<index>是相对于方法起始处的字节偏移量。<opcode>为指令的操作码的助记符号， <operandN>是指令的操作数，一条指令可以有 0 至多个操作数。 <comment>为行尾的语法注释， 譬如：

```java
8 bipush 100 // Push int constant 100
```

注释中的某些部分由 javap 自动加入， 其余部分由作者手工添加。每条指令之前的<index>可以作为控制转移指令（ Control Transfer Instruction） 的跳转目标。 譬如： goto 8 指令表示跳转到索引为 8 的指令上继续执行。需要注意的是， Java 虚拟机的控制转移指令的实际操作数是在当前指令的操作码集合中的地址偏移量，但这些操作数会被 javap 工具（以及在本章的内容中） 按照更容易被人所阅读的方式来显示。
在每一行中，在表示运行时常量池索引的操作数前，会井号（ ’#’）开头，在指令后的注释中，会带有这个操作数的描述， 譬如：

```java
10 ldc #1 // Push float constant 100.0
```

和

```java
9 invokevirtual #4 // Method Example.addTwo(II)I
```

本章节主要目的是描述虚拟机的编译过程， 我们将忽略一些诸如操作数容量等的细节问题。 













