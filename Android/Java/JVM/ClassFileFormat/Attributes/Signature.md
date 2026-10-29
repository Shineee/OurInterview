# Signature 属性

Signature 属性是可选的定长属性，位于 ClassFile（ §4.1）， field_info（ §4.5）或 method_info（ §4.6）结构的属性表中。在 Java 语言中，任何类、 接口、 初始化方法或成员的泛型签名如果包含了类型变量（ Type Variables） 或参数化类型（ ParameterizedTypes），则 Signature 属性会为它记录泛型签名信息。
Signature 属性格式如下：

```
Signature_attribute {

u2 attribute_name_index;

u4 attribute_length;

u2 signature_index;

}
```

Signature_attribute 结构各项的说明如下：

* attribute_name_index

  attribute_name_index 项的值必须是一个对常量池的有效索引。常量池在索引处的成员必须是 CONSTANT_Utf8_info（ §4.4.7）结构，表示字符串“ Signature”。

* attribute_length

  signature_attribute 结构的 attribute_length 项的值必须为 2。

* signature_index

  signature_index 项的值必须是一个对常量池的有效索引。 常量池在该索引处的项必须是 CONSTANT_Utf8_info（ §4.4.7）结构，表示类签名或方法类型签名或字段类型签名：如果当前的 Signature 属性是 ClassFile 结构的属性，则这个结构表示类签名，如果当前的 Signature 属性是 method_info 结构的属性，则这个结构表示方法类型签名，如果当前 Signature 属性是 field_info 结构的属性，则这个结构表示字段类型签名。 














