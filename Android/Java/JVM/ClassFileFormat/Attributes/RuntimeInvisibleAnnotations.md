# RuntimeInvisibleAnnotations 属性

RuntimeInvisibleAnnotations 属性和 RuntimeVisibleAnnotations 属性相似，不同的是 RuntimeVisibleAnnotations 表示的注解不能被反射 API 访问，除非 Java 虚拟机通过特殊的实现相关的方式（ 譬如特定的命令行参数）收到才会（为反射的 API） 使用这些注解。否则， Java 虚拟机将忽略 RuntimeVisibleAnnotations 属性。 

RuntimeInvisibleAnnotations 属性是一个变长属性，位于 ClassFile（ §4.1） ,field_info（ §4.5）或method_info（ §4.6）结构的属性表中。用于保存 Java 语言中的类、字段或方法的运行时的非可见注解。每个 ClassFile， field_info 和 method_info 结构最多只能含有一个 RuntimeInvisibleAnnotations 属性，它为当前的程序元素保存所有的运行时非可见的 Java 语言注解。
RuntimeInvisibleAnnotations 属性格式如下：

```
RuntimeInvisibleAnnotations_attribute {

u2 attribute_name_index;

u4 attribute_length;

u2 num_annotations;

annotation annotations[num_annotations];

}
```

RuntimeInvisibleAnnotations_attribute 结构各项的说明如下：

* attribute_name_index

  attribute_name_index 项的值必须是对常量池的一个有效索引。常量池在该索引处的成员必须是 CONSTANT_Utf8_info（ §4.4.7）结构，表示字符串“ RuntimeInvisibleAnnotations”。

* attribute_length

  attribute_length 项的值给出了当前属性的长度，不包括开始的 6 个字节。attribute_length 项的值由当前结构的运行时非可见注解的数量和值决定。

* num_annotations

  num_annotations 项的值给出了当前结构表示的运行时可见注解的数量。每个程序元素最多可能会被附加 65535 个运行时非可见的 java 语言注解。

* annotations[]

  annotations[]数组的每个成员的值表示一个程序元素的唯一的运行时可见注解。 