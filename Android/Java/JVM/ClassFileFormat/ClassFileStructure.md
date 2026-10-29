# ClassFile 结构


每一个 Class 文件对应于一个如下所示的 ClassFile 结构体。

```java
ClassFile {

u4 magic;

u2 minor_version;

u2 major_version;

u2 constant_pool_count;

cp_info constant_pool[constant_pool_count-1];

u2 access_flags;

u2 this_class;

u2 super_class;

u2 interfaces_count;

u2 interfaces[interfaces_count];

u2 fields_count;

field_info fields[fields_count];

u2 methods_count;

method_info methods[methods_count];

u2 attributes_count;

attribute_info attributes[attributes_count];

}
```

ClassFile 结构体中，各项的含义描述如下：

* magic

  魔数， 魔数的唯一作用是确定这个文件是否为一个能被虚拟机所接受的 Class 文件。 魔数值固定为 0xCAFEBABE， 不会改变。

* minor_version、 major_version

  副版本号和主版本号， minor_version 和 major_version 的值分别表示 Class 文件的副、 主版本。 它们共同构成了 Class 文件的格式版本号。 譬如某个 Class 文件的主版本号为 M，副版本号为 m，那么这个 Class 文件的格式版本号就确定为 M.m。 Class 文件格式版本号大小的顺序为： 1.5 < 2.0 < 2.1。

  一个 Java 虚拟机实例只能支持特定范围内的主版本号（ Mi 至 Mj） 和 0 至特定范围内 （ 0 至 m）的副版本号。假设一个 Class 文件的格式版本号为 V， 仅当 Mi.0 ≤ v ≤ Mj.m成立时， 这个 Class 文件才可以被此 Java 虚拟机支持。不同版本的 Java 虚拟机实现支持的版本号也不同，高版本号的 Java 虚拟机实现可以支持低版本号的 Class 文件， 反之则不成立①。

* constant_pool_count

  常量池计数器，constant_pool_count 的值等于 constant_pool 表中的成员数加 1。constant_pool 表的索引值只有在大于 0 且小于 constant_pool_count 时才会被认为是有效的②， 对于 long 和 double 类型有例外情况，可参见§4.4.5。

* constant_pool[]

  常量池， constant_pool 是一种表结构（ §4.4）， 它包含 Class 文件结构及其子结构中引用的所有字符串常量、 类或接口名、字段名和其它常量。 常量池中的每一项都具备相同的格式特征——第一个字节作为类型标记用于识别该项是哪种类型的常量，称为“ tag byte”。 常量池的索引范围是 1 constant_pool_count-1。

* access_flags

  访问标志， access_flags 是一种掩码标志， 用于表示某个类或者接口的访问权限及基础属性。 access_flags 的取值范围和相应含义见表 4.1 所示。

| 标记名            | 值      | 含义                               |
| -------------- | ------ | -------------------------------- |
| ACC_PUBLIC     | 0x0001 | 可以被包的类外访问。                       |
| ACC_FINAL      | 0x0010 | 不允许有子类。                          |
| ACC_SUPER      | 0x0020 | 当用到invokespecial指令时，需要特殊处理的父类方法。 |
| ACC_INTERFACE  | 0x0200 | 标识定义的是接口而不是类。                    |
| ACC_ABSTRACT   | 0x0400 | 不能被实例化。                          |
| ACC_SYNTHETIC  | 0x1000 | 标识并非Java源码生成的代码。                 |
| ACC_ANNOTATION | 0x2000 | 标识注解类型。                          |
| ACC_ENUM       | 0x4000 | 标识枚举类型。                          |

带有 ACC_SYNTHETIC 标志的类， 意味着它是由编译器自己产生的而不是由程序员编写的源代码生成的。

带有 ACC_ENUM 标志的类， 意味着它或它的父类被声明为枚举类型。

带有 ACC_INTERFACE 标志的类，意味着它是接口而不是类，反之是类而不是接口。如果一个 Class 文件被设置了 ACC_INTERFACE 标志，那么同时也得设置ACC_ABSTRACT 标志（ JLS §9.1.1.1）。同时它不能再设置 ACC_FINAL、ACC_SUPER 和 ACC_ENUM 标志。
注解类型必定带有 ACC_ANNOTATION 标记，如果设置了 ANNOTATION 标记，ACC_INTERFACE 也必须被同时设置。如果没有同时设置 ACC_INTERFACE 标记，那么这个Class文件可以具有表4.1中的除ACC_ANNOTATION外的所有其它标记。当然 ACC_FINAL 和 ACC_ABSTRACT 这类互斥的标记除外（ JLS §8.1.1.2）。
ACC_SUPER 标志用于确定该 Class 文件里面的 invokespecial 指令使用的是哪一种执行语义。目前 Java 虚拟机的编译器都应当设置这个标志。 ACC_SUPER 标记是为了向后兼容旧编译器编译的 Class 文件而存在的，在 JDK1.0.2 版本以前的编译器产生的 Class 文件中， access_flag 里面没有 ACC_SUPER 标志。同时，JDK1.0.2 前的 Java 虚拟机遇到 ACC_SUPER 标记会自动忽略它。
在表 4.1 中没有使用的 access_flags 标志位是为未来扩充而预留的，这些预留的标志为在编译器中会被设置为 0， Java 虚拟机实现也会自动忽略它们。

* this_class

  类索引， this_class 的值必须是对 constant_pool 表中项目的一个有效索引值。constant_pool 表在这个索引处的项必须为 CONSTANT_Class_info 类型常量（ §4.4.1），表示这个 Class 文件所定义的类或接口。

* super_class

  父类索引，对于类来说， super_class 的值必须为 0 或者是对 constant_pool 表中项目的一个有效索引值。 如果它的值不为 0，那 constant_pool 表在这个索引处的项必须为 CONSTANT_Class_info 类型常量（ §4.4.1），表示这个 Class 文件所定义的类的直接父类。 当前类的直接父类，以及它所有间接父类的 access_flag 中都不能带 有 ACC_FINAL 标记。 对于接口来说， 它的 Class 文件的 super_class 项的值必须是对 constant_pool 表中项目的一个有效索引值。 constant_pool 表在这个索引处的项必须为代表 java.lang.Object 的 CONSTANT_Class_info 类型常量 （ §4.4.1）。如果 Class 文件的 super_class 的值为 0，那这个 Class 文件只可能是定义的是java.lang.Object 类，只有它是唯一没有父类的类。

* interfaces_count

  接口计数器， interfaces_count 的值表示当前类或接口的直接父接口数量。

* interfaces[]

  接口表， interfaces[]数组中的每个成员的值必须是一个对 constant_pool 表中项目的一个有效索引值， 它的长度为 interfaces_count。每个成员 interfaces[i] 必须为 CONSTANT_Class_info 类型常量（ §4.4.1），其中 0 ≤ i <interfaces_count。 在 interfaces[]数组中，成员所表示的接口顺序和对应的源代码中给定的接口顺序（ 从左至右） 一样，即 interfaces[0]对应的是源代码中最左边的接口。

* fields_count

  字段计数器， fields_count 的值表示当前 Class 文件 fields[]数组的成员个数。fields[]数组中每一项都是一个 field_info 结构（ §4.5）的数据项， 它用于表示该类或接口声明的类字段或者实例字段①。

* fields[]

  字段表， fields[]数组中的每个成员都必须是一个 fields_info 结构（ §4.5）的数据项，用于表示当前类或接口中某个字段的完整描述。 fields[]数组描述当前类或接口声明的所有字段，但不包括从父类或父接口继承的部分。

* methods_count

  方法计数器， methods_count 的值表示当前 Class 文件 methods[]数组的成员个数。Methods[]数组中每一项都是一个 method_info 结构（ §4.5）的数据项。

* methods[]

  方法表， methods[]数组中的每个成员都必须是一个 method_info 结构（ §4.6）的数据项，用于表示当前类或接口中某个方法的完整描述。如果某个 method_info 结构的 access_flags 项既没有设置 ACC_NATIVE 标志也没有设置 ACC_ABSTRACT 标志，那么它所对应的方法体就应当可以被 Java 虚拟机直接从当前类加载，而不需要引用其它类。 method_info 结构可以表示类和接口中定义的所有方法，包括实例方法、类方法、实例初始化方法方法（ §2.9） 和类或接口初始化方法方法（ §2.9）。 methods[]数组只描述当前类或接口中声明的方法，不包括从父类或父接口继承的方法。

* attributes_count

  属性计数器， attributes_count 的值表示当前 Class 文件 attributes 表的成员个数。 attributes 表中每一项都是一个 attribute_info 结构（ §4.7）的数据项。

* attributes[]

  属性表， attributes 表的每个项的值必须是 attribute_info 结构（ §4.7）。 在本规范里， Class 文件结构中的 attributes 表的项包括下列定义的属性：InnerClasses （ §4.7.6）、EnclosingMethod （ §4.7.7）、Synthetic （ §4.7.8）、Signature（ §4.7.9）、 SourceFile（ §4.7.10）， SourceDebugExtension（ §4.7.11）、 Deprecated（ §4.7.15）、 RuntimeVisibleAnnotations（ §4.7.16）、 RuntimeInvisibleAnnotations（ §4.7.17）以及BootstrapMethods（ §4.7.21）属性。对于支持 Class 文件格式版本号为 49.0 或更高的 Java 虚拟机实现，必须正确识别并读取 attributes 表中的 Signature（ §4.7.9）、 RuntimeVisibleAnnotations（ §4.7.16） 和RuntimeInvisibleAnnotations（ §4.7.17） 属性。 对于支持 Class 文件格式版本号为 51.0 或更高的 Java 虚拟机实现，必须正确识别并读取 attributes 表中的BootstrapMethods（ §4.7.21） 属性。 本规范要求任一 Java 虚拟机实现可以自动忽略 Class 文件的 attributes 表中的若干（ 甚至全部） 它不可识别的属性项。任何本规范未定义的属性不能影响 Class 文件的语义，只能提供附加的描述信息（ §4.7.1）。




