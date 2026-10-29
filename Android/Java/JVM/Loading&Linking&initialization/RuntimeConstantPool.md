# 运行时常量池

Java 虚拟机为每个类型都维护一个常量池（ §2.5.5）。它是 Java 虚拟机中的运行时数据结构，像传统编程语言实现中的符号表一样有很多用途。
当类或接口创建时（ §5.3），它的二进制表示中的 constant_pool 表（ §4.4）被用来构造运行时常量池。运行时常量池中的所有引用最初都是符号引用。这些符号引用来自于类或接口的二进制表示的如下结构中：

* 某个类或接口的符号引用来自于类或接口二进制表示中的 CONSTANT_Class_info 结构（ §4.4）。这种引用提供的类或接口的名称如同 Class.getName()方法返回值的格式一样，也就是说：

  * 对于非数组的类或接口，那就是类或接口的二进制名称（ §4.2.1）。
  * 对于一个 M 维的数组类，名称会以 M 个 ASCII 字符"["开头，随后是数组元素类型的表示：
    * 如果数组的元素类型是 Java 原始类型之一，它就以相应的字段描述符（ §4.3.2）来表示。
    * 否则，如果数组元素类型是某种引用类型，它就以 ASCII 字符"L"加上二进制名称，并以 ASCII 字符";"结尾的字符串表示。

* 在本章中，无论何时提到类或接口的名称，读者都可以按 Class.getName 方法的返回值的格式来理解。

* 类或接口的某个字段的符号引用来自于类或接口二进制表示中的CONSTANT_Fieldref_info 结构（ §4.4.2）。这种引用包含了字段的名称和描述符，及指向字段所属类或接口的符号引用。

* 类中某个方法的符号引用来自于类或接口二进制表示中的 CONSTANT_Methodref_info 结构（ §4.4.2）。这种引用提供方法的名称和描述符，及指向方法所属类的符号引用。

* 接口的某个方法的符号引用来自于类或接口二进制表示中的CONSTANT_InterfaceMethodref_info 结构（ §4.4.2）。这种引用提供接口方法的名称和描述符，及指向方法所属接口的符号引用。

* 方法句柄（ Method Handle）的符号引用来自于类或接口二进制表示中的CONSTANT_MethodHandle_info 结构（ §4.4.8）。

* 方法类型（ Method Type）的符号引用来自于类或接口二进制表示中的CONSTANT_MethodType_info 结构（ §4.4.9）。

* 调用点限定符（ Call Site Specifier）的符号引用来自于类或接口二进制表示的CONSTANT_InvokeDynamic_info 结构（ §4.4.10）。这种引用包含了：

  - 方法句柄的符号引用，为 invokedynamic 指令的引导方法（ Bootstrap Method）来提供服务。
  -  一系列符号引用（到类、方法类型和方法句柄）、字符常量和运行时常量（如 Java 原生数值类型的值），它们将作为静态参数（ Static Arguments）提供给引导方法。
  - 调用方法的名称与描述符

  另外，一些运行时数值它不是由符号引用，而是由 constant_pool 表中某些项的值来决定：

* 字符常量表示 String 类实例的一个引用，它来自于类或接口二进制表示的CONSTANT_String_info 结构（ §4.4.3）。 CONSTANT_String_info 结构包含了由Unicode 码点（ Code points） ①序列来组成字符常量。Java 语言需要全局统一的字符常量（这就意味着如果不同字面量（ Literal）包含着相同的码点序列，就必须引用着相同的 String 类的实例（ JLS §3.10.5））。此外，在任意字符串上调用 String.intern 方法，如果那个字符串是字面量的话，方法的结果应当是对相同字面量的 String 实例的引用。因此，

  ```
  ("a" + "b" + "c").intern() == "abc"
  ```

  必须返回 true。

  为了得到字符常量， Java 虚拟机需要检查 CONSTANT_String_info 结构中的码点序列。

  * 如果 String.intern 方法以前曾经被某个与 CONSTANT_String_info 结构中的码点序列一样的 String 实例调用过，那么此次字符常量获取的结果将是一个指向相同String 实例的引用。
  * 否则，一个新的 String 实例会被创建，它会包含着指定 CONSTANT_String_info结构中 Unicode 码点序列；字符常量获取的结果是指向那个新 String 实例的引用。最后，新 String 实例的 intern 方法被 Java 虚拟机自动调用。

* 其它运行时常量值来自于类或接口二进制表示的 CONSTANT_Interger_info、CONSTANT_Float_info、 CONSTANT_Long_info 或是 CONSTANT_Double_info 结构（ §4.4.4， §4.4.5）。请注意，这里 CONSTANT_Float_info 结构的值以 IEEE 754 单精度浮点格式表示， CONSTANT_Double_info 结构的值以 IEEE 754 双精度浮点格式表示（ §4.4.4， §4.4.5）。来自这些结构的运行时常量值必须可以用 IEEE 754 单或双精度浮点格式分别表示。

  在类或接口的二进制表示中， constant_pool 表中剩下的结构还有CONSTANT_NameAndType_info（ §4.4.6） 和 CONSTANT_Utf8_info（ §4.4.7），它们被间接用以获得对类、接口、方法、字段、方法类型和方法句柄的符号引用，或是在需要得到字符常量和调用点限定符时被引用。 