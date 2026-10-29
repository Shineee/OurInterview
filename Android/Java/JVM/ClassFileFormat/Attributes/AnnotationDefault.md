# AnnotationDefault 属性

AnnotationDefault 属性是一个变长属性，位于某些特殊的 method_info（ §4.6）结构的属性表中，这些结构表示注解类型的元素。 AnnotationDefault 属性用于保存method_info 结构表示的元素的默认值。每个 method_info 结构表示的一个元素的注解最多只能有一个 AnnotationDefault 属性。 Java 虚拟机必须保证这些默认值可见，可供反射 API 使用。
AnnotationDefault 属性格式如下：

```
AnnotationDefault_attribute {

u2 attribute_name_index;

u4 attribute_length;

element_value default_value;

} 
```

AnnotationDefault_attribute 结构各项的说明如下：

* attribute_name_index

  attribute_name_index 项的值必须是对常量池的一个有效索引。常量池在该索引处的成员必须是 CONSTANT_Utf8_info（ §4.4.7）结构，表示字符串“ AnnotationDefault”。

* attribute_length

  attribute_length 项的值给出了当前属性的长度，不包括开始的 6 个字节。

  attribute_length 项的值由默认值决定。

* default_value

  default_value 项表示 AnnotationDefault 属性所对应注解类型元素的默认值。 

