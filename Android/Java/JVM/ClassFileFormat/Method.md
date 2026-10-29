# 方法

所有方法 （ Method），包括实例初始化方法和类初始化方法 （ §2.9）在内，都由 method_info结构所定义。在一个 Class 文件中，不会有两个方法同时具有相同的方法名和描述符（ § 4.3.3）。
method_info 结构格式如下：

```
method_info {

u2 access_flags;

u2 name_index;

u2 descriptor_index;

u2 attributes_count;

attribute_info attributes[attributes_count];

}
```

method_info 结构各项的说明如下：

* access_flags

  access_flags 项的值是用于定义当前方法的访问权限和基本属性的掩码标志，access_flags 的取值范围和相应含义见表 4.5 所示。

  表 4.5 方法 access_flags 标记列表

|       标记名        |   值    |             说明             |
| :--------------: | :----: | :------------------------: |
|    ACC_PUBLIC    | 0x0001 |    public，表示字段可以从任何包访问。    |
|   ACC_PRIVATE    | 0x0002 |   private，表示字段仅能该类自身调用。    |
|  ACC_PROTECTED   | 0x0004 |   protected，表示字段可以被子类调用。   |
|    ACC_STATIC    | 0x0008 |       static，表示静态字段。       |
|    ACC_FINAL     | 0x0010 |    final，表示字段定义后值无法修改。     |
| ACC_SYNCHRONIZED | 0x0020 |   synchronized，方法由管程同步。    |
|    ACC_BRIDGE    | 0x0040 |      bridge，方法由编译器产生。      |
|   ACC_VARARGS    | 0x0080 |        表示方法带有变长参数。         |
|    ACC_NATIVE    | 0x0100 |  native，方法引用非java语言的本地方法。  |
|   ACC_ABSTRACT   | 0x0400 |     abstract，方法没有具体实现      |
|    ACC_STRICT    | 0x0800 | strictfp，方法使用FP-strict浮点格式 |
|  ACC_SYNTHETIC   | 0x1000 |     方法在源文件中不出现，由编译器产生      |

ACC_VARARGS 标志是用于说明方法在源码层的参数列表是否变长的。如果是变长的，则在编译时，方法的 ACC_VARARGS 标志设置 1，其余的方法 ACC_VARARGS 标志设置为 0。ACC_BRIDGE 标志用于说明这个方法是由编译生成的桥接方法①。
如果方法设置了 ACC_SYNTHETIC 标志，怎说明这个方法是由编译器生成的并且不会在源代码中出现， 少量的例外情况将在 4.7.8 节“ Synthetic 属性” 中提到。
Class 文件中的方法可以设置多个表 4.5 中的标志，但是有些标志是互斥的：一个方法只能设置 ACC_PRIVATE， ACC_PROTECTED 和 ACC_PUBLI 三个标志中的一个标志；如果一个方法被设置 ACC_ABSTRACT 标志，则这个方法不能被设置 ACC_FINAL，ACC_NATIVE, ACC_PRIVATE, ACC_STATIC, ACC_STRICT 和 ACC_SYNCHRONIZED
标志（ JLS §8.4.3.1, JLS §8.4.3.3, JLS §8.4.3.4）。
所有的接口方法必须被设置 ACC_ABSTRACT 和 ACC_PUBLIC 标志；还可以选择设置ACC_VARARGS，ACC_BRIDGE 和 ACC_SYNTHETIC 标志，但是不能再设置表 4.5 中其它的标志了（ JLS §9.4）。
实例初始化方法（ §2.9）只能被 ACC_PRIVATE， ACC_PROTECTED 和 ACC_PUBLIC中的一个标志；还可以设置 ACC_STRICT， ACC_VARARGS 和 ACC_SYNTHETIC 标志， 但是不能再设置表 4.5 中的其它标志了。
类初始化方法（ §2.9） 由 Java 虚拟机隐式自动调用，它的 access_flags 项的值除了 ACC_STRICT 标志，其它的标志都将被忽略。
在表 4.5 中没有出现的 access_flags 项值为未来扩充而预留，在生成的 Class 文件中应被设置成 0， Java 虚拟机实现应该忽略它们。

* name_index

  name_index 项的值必须是对常量池的一个有效索引。 常量池在该索引处的项必须是CONSTANT_Utf8_info（ §4.4.7）结构，它要么表示初始化方法的名字（ <init>或<clinit>），要么表示一个方法的有效的非全限定名（ §4.2.2）

* descriptor_index

  descriptor_index 项的值必须是对常量池的一个有效索引。 常量池在该索引处的项必须是 CONSTANT_Utf8_info （ §4.4.7）结构，表示一个有效的方法的描述符（ §4.3.3）

  注意：本规范在未来的某个版本中可能会要求当 access_flags 项的 ACC_VARARGS 标志被设置时， 方法描述符中的最后一个参数必须是数组类型。

* attributes_count

  attributes_count 的项的值表示这个方法的附加属性（ §4.7）的数量。

* attributes[]

  attributes 表的每一个成员的值必须是 attribute（ §4.7）结构， 一个方法可以有任意个与之相关的属性。本规范所定义的 method_info 结构中，属性表可出现的成员有： Code（ §4.7.3），Exceptions（ §4.7.5）， Synthetic（ §4.7.8）， Signature（ §4.7.9），Deprecated（ §4.7.15）， untimeVisibleAnnotations（ §4.7.16），RuntimeInvisibleAnnotations（ §4.7.17），RuntimeVisibleParameterAnnotations（ §4.7.18），RuntimeInvisibleParameterAnnotations（ §4.7.19）和AnnotationDefault（ §4.7.20）结构。所有 Java 虚拟机实现必须默认忽略 method_info 结构中 attributes 表所不可识别的成员。本规范中没有定义的属性不可影响 Class 文件的语义， 它们只能提供附加描述信息（ §4.7.1）。 





