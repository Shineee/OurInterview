# 字段

每个字段（ Field） 都由 field_info 结构所定义，在同一个 Class 文件中，不会有两个字段同时具有相同的字段名和描述符（ §4.3.2）。
field_info 结构格式如下：

```
field_info {

u2 access_flags;

u2 name_index;

u2 descriptor_index;

u2 attributes_count;

attribute_info attributes[attributes_count];

}
```

field_info 结构各项的说明如下：

* access_flags
  access_flags 项的值是用于定义字段被访问权限和基础属性的掩码标志。

  access_flags 的取值范围和相应含义见表 4.4 所示。

  表 4.4 字段 access_flags 标记列表

  |      标记名      |   值    |           说明           |
  | :-----------: | :----: | :--------------------: |
  |  ACC_PUBLIC   | 0x0001 |  public，表示字段可以从任何包访问。  |
  |  ACC_PRIVATE  | 0x0002 | private，表示字段仅能该类自身调用。  |
  | ACC_PROTECTED | 0x0004 | protected，表示字段可以被子类调用。 |
  |  ACC_STATIC   | 0x0008 |     static，表示静态字段。     |
  |   ACC_FINAL   | 0x0010 |  final，表示字段定义后值无法修改。   |
  | ACC_VOLATILE  | 0x0040 |   volatile，表示字段是易变的。   |
  | ACC_TRANSIENT | 0x0080 | transient，表示字段不会被序列化。  |
  | ACC_SYNTHETIC | 0x1000 |     表示字段由编译器自动产生。      |
  |   ACC_ENUM    | 0x4000 |    enum，表示字段为枚举类型。     |

字段如果带有 ACC_SYNTHETIC 标志，则说明这个字段不是由源码产生的，而是由编译器自动产生的。
字段如果被标有 ACC_ENUM 标志，这说明这个字段是一个枚举类型成员。Class 文件中的字段可以被设置多个表 4.4 的标记。不过有些标记是互斥的，一个字段最多只能设置 ACC_PRIVATE， ACC_PROTECTED，和 ACC_PUBLIC（ JLS §8.3.1）三个标志中的一个，也不能同时设置标志 ACC_FINAL 和 ACC_VOLATILE（ JLS §
8.3.1.4）。
接口中的所有字段都具有 ACC_PUBLIC， ACC_STATIC 和 ACC_FINAL 标记，也可能被设置 ACC_SYNTHETIC 标记，但是不能含有表 4.4 中的其它符号标记了（ JLS §9.3）。在表 4.4 中没有出现的 access_flags 项的值为扩充而预留，在生成的 Class 文件中应被设置成 0， Java 虚拟机实现也应该忽略它们。

* name_index

  name_index 项的值必须是对常量池的一个有效索引。 常量池在该索引处的项必须是

  CONSTANT_Utf8_info（ §4.4.7）结构，表示一个有效的字段的非全限定名（ §4.2.2）。

* descriptor_index

  descriptor_index 项的值必须是对常量池的一个有效索引。 常量池在该索引处的项必须是 CONSTANT_Utf8_info（ §4.4.7）结构，表示一个有效的字段的描述符（ §4.3.2）。

* attributes_count

  attributes_count 的项的值表示当前字段的附加属性（ §4.7）的数量。

* attributes[]

  attributes 表的每一个成员的值必须是 attribute（ §4.7）结构，一个字段可以有任意个关联属性。

  本规范所定义的 field_info 结构中， attributes 表可出现的成员有： 

  Deprecated（ §4.7.15） , RuntimeVisibleAnnotations（ §4.7.16） 和RuntimeInvisibleAnnotations（ §4.7.17）。

  Java 虚拟机实现必须正确的识别和读取 field_info 结构的 attributes 表中的ConstantValue（ §4.7.2）属性。如果 Java 虚拟机实现支持版本号为 49.0 或更高的 Class 文件，那么它必须正确的识别和读取这些 Class 文件中的 Signature（ §4.7.9） , RuntimeVisibleAnnotations（ §4.7.16） 和RuntimeInvisibleAnnotations（ §4.7.17）结构。

  所有 Java 虚拟机实现都必须默认忽略 field_info 结构中 attributes 表所不可识别的成员。本规范中没有定义的属性不可影响 Class 文件的语义， 它们只能提供附加描述信息（ §4.7.1）。 




