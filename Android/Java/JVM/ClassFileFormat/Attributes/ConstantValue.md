# ConstantValue 属性


ConstantValue 属性是定长属性，位于 field_info（ §4.5）结构的属性表中。ConstantValue 属性表示一个常量字段的值。在一个 field_info 结构的属性表中最多只能有一个 ConstantValue 属性。如果该字段为静态类型（即 field_info 结构的access_flags项（ 表 4.4）设置了 ACC_STATIC 标志），则说明这个 field_info 结构表示的常量字段值将被分配为它的 ConstantValue 属性表示的值，这个过程也是类或接口申明的常量字段（ ConstantField（ §5.5））初始化的一部分。这个过程发生在引用类或接口的类初始化方法（ §2.9） 执行之前。
如果 field_info 结构表示的非静态字段包含了 ConstantValue 属性，那么这个属性必须被虚拟机所忽略。所有 Java 虚拟机实现必须能够识别 ConstantValue 属性。
ConstantValue 属性的格式如下：

```
ConstantValue_attribute {

u2 attribute_name_index;

u4 attribute_length;

u2 constantvalue_index;

}
```

ConstantValue 结构各项的说明如下：

* attribute_name_index

  attribute_name_index 项的值，必须是一个对常量池的有效索引。 常量池在该索引处的项必须是 CONSTANT_Utf8_info（ §4.4.7）结构，表示字符串“ ConstantValue”。

* attribute_length

  ConstantValue_attribute 结构的 attribute_length 项的值固定为 2。

* constantvalue_index

  constantvalue_index 项的值，必须是一个对常量池的有效索引。 常量池在该索引处的项给出该属性表示的常量值。 常量池的项的类型表示的字段类型如表 4.7 所示。

表 4.7 ConstantValue 属性的类型 

|            字段类型             |       项目类型       |
| :-------------------------: | :--------------: |
|            long             |  CONSTANT_Long   |
|            float            |  CONSTANT_Float  |
|           double            | CONSTANT_Double  |
| int，short，char，byte，boolean | CONSTANT_Integer |
|           String            | CONSTANT_String  |