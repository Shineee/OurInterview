# InnerClasses 属性

InnerClasses 属性是一个变长属性，位于 ClassFile（ §4.1）结构的属性表。本小结为了方便说明特别定义一个表示类或接口的 Class 格式为 C。如果 C 的常量池中包含某个CONSTANT_Class_info 成员，且这个成员所表示的类或接口不属于任何一个包，那么 C 的ClassFile 结构的属性表中就必须含有对应的 InnerClasses 属性。InnerClasses 属性是在 JDK 1.1 中为了支持内部类和内部接口而引入的。

```
InnerClasses_attribute {

u2 attribute_name_index;

u4 attribute_length;

u2 number_of_classes;

{ u2 inner_class_info_index;

u2 outer_class_info_index;

u2 inner_name_index;

u2 inner_class_access_flags;

} classes[number_of_classes];

}
```

* attribute_name_index

  attribute_name_index 项的值必须是一个对常量池的有效引用。 常量池在该索引处的成员必须是 CONSTANT_Utf8_info （ §4.4.7）结构，表示字符串"InnerClasses"。

* attribute_length

  attribute_length 项的值给出了这个属性的长度，不包括前 6 个字节。

* number_of_classes

  number_of_classes 项的值表示 classes[]数组的成员数量。

* classes[]常量池中的每个 CONSTANT_Class_info 结构如果表示的类或接口并非某个包的成员，则每个类或接口在 classes[]数组中都有一个成员与之对应。

如果 Class 中包含某些类或者接口，那么它的常量池（以及对应的 InnerClasses 属性）必须包含这些成员，即使某些类或者接口没有被这个 Class 使用过。
这条规则暗示着内部类或内部接口成员在其被定义的外部类（ Enclosing Class）中都会包含有它们的 InnerClasses 信息。
classes[]数组中每个成员包含以下 4 个项：

* inner_class_info_index

  inner_class_info_index 项的值必须是一个对常量池的有效索引。 常量池在该索引处的项必须是 CONSTANT_Class_info（ §4.4.1）结构，表示接口 C。当前元素的另外 3 项都用于描述 C 的信息。

* outer_class_info_index

  如果 C 不是类或接口的成员（ 也就是 C 为顶层类或接口（ JLS §7.6）、 局部类（ JLS §14.3）或匿名类（ JLS §15.9.5）） ,那么 outer_class_info_index 项的值为 0，否则这个项的值必须是对常量池的一个有效索引， 常量池在该索引处的项必须是CONSTANT_Class_info（ §4.4.1）结构，代表一个类或接口， C 为这个类或接口的成员。

* inner_name_index

  如果 C 是匿名类（ JLS §15.9.5）， inner_name_index 项的值则必须为 0。否则这个项的值必须是对常量池的一个有效索引， 常量池在该索引处的项必须CONSTANT_Utf8_info（ §4.4.7）结构，表示 C 的 Class 文件在对应的源文件中定义的 C 的原始简单名称（ Original Simple Name）

* inner_class_access_flags

  inner_class_access_flags 项的值是一个掩码标志，用于定义 Class 文件对应的源文件中 C 的访问权和基本属性。 用于编译器在无法访问源文件时可以恢复 C 的原始信息。 inner_class_access_flags 项的取值范围和说明见表 4.8。

表 4.8 内部类访问全和基础属性标志

|      标记名       |   值    |         含义         |
| :------------: | :----: | :----------------: |
|   ACC_PUBLIC   | 0x0001 |    源文件定义public     |
|  ACC_PRIVATE   | 0x0002 |    源文件定义private    |
| ACC_PROTECTED  | 0x0004 |   源文件定义protected   |
|   ACC_STATIC   | 0x0008 |    源文件定义static     |
|   ACC_FINAL    | 0x0010 |     源文件定义final     |
| ACC_INTERFACE  | 0x0200 |   源文件定义interface   |
|  ACC_ABSTRACT  | 0x0400 |   源文件定义abstract    |
| ACC_SYNTHETIC  | 0x1000 | 声明synthetic，非源文件定义 |
| ACC_ANNOTATION | 0x2000 |    声明annotation    |
|    ACC_ENUM    | 0x4000 |       声明enum       |

所有表 4.8 中没有定义的 inner_class_access_flags 项，都是为未来使用而预留 的。这些字节在通常的 Class 文件中应被设置为 0， Java 虚拟机实现应忽略它们。

如果 Class 文件的版本号为 51.0 或更高，属性表中有 InnerClasses 属性， 并且InnerClasses 属性的 classes[]数组中的 inner_name_index 项的值为 0，则它对应的outer_class_info_index 项的值也必须为 0。

Oracle 的 Java 虚拟机实现不会检查 InnerClasses 属性和某个属性引用的类或接口的Class 文件的一致性。 










