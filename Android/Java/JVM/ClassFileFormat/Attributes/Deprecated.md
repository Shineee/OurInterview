# Deprecated 属性


Deprecated 属性是可选定长属性，位于 ClassFile（ §4.1） , field_info（ §4.5）
或 method_info（ §4.6）结构的属性表中。类、接口、方法或字段都可以带有为 Deprecated属性，如果类、接口、方法或字段标记了此属性， 则说明它将会在后续某个版本中被取代。在运行时解释器或工具（ 譬如编译器）读取 Class 文件格式时，可以用 Deprecated 属性来告诉使用者避免使用这些类、接口、方法或字段，选择其他更好的方式。 Deprecated 属性的出现不会修改类或接口的语义。
Deprecated 属性是在 JDK 1.1 为了支持注释中的关键词@deprecated 而引入的。
Deprecated 属性格式如下：

```
Deprecated_attribute {

u2 attribute_name_index;

u4 attribute_length;

}
```

Deprecated_attribute 结构各项的说明如下：

* attribute_name_index

  attribute_name_index 项的值必须是对常量池的一个有效索引。常量池在该索引处的成员必须是 CONSTANT_Utf8_info（ §4.4.7） 结构，表示字符串“ Deprecated”。

* attribute_length

  attribute_length 项的值固定为 0。 







