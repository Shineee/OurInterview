# RuntimeVisibleAnnotations 属性

RuntimeVisibleAnnotations 属性是可变长属性，位于 ClassFile（ §4.1），field_info（ §4.5）或 method_info（ §4.6）结构的属性表中。
RuntimeVisibleAnnotations 用于保存 Java 语言中的类、字段或方法的运行时的可见注解（ Annotations）。每个 ClassFile， field_info 和 method_info 结构最多只能含有一个RuntimeVisibleAnnotations 属性为当前的程序元素保存所有的运行时可见的 Java 语言注 解。 Java 虚拟机必须支持这些注解可被反射的 API 使用它们。

RuntimeVisibleAnnotations 属性格式如下：

```
RuntimeVisibleAnnotations_attribute {

u2 attribute_name_index;

u4 attribute_length;

u2 num_annotations;

annotation annotations[num_annotations];

}
```

RuntimeVisibleAnnotations_attribute 结构各项的说明如下：

* attribute_name_index

  attribute_name_index 项的值必须是对常量池的一个有效索引。常量池在该索引处的成员必须是 CONSTANT_Utf8_info（ §4.4.7）结构，表示字符串“ RuntimeVisibleAnnotations”。


* attribute_length

  attribute_length 项的值给出了当前属性的长度，不包括开始的 6 个字节。attribute_length 项的值由当前结构的运行时可见注解的数量和值决定。

* num_annotations

  num_annotations 项的值给出了当前结构表示的运行时可见注解的数量。每个程序元素最多可能会被附加 65535 个运行时可见的 java 语言注解。

* annotations[]

  annotations[]数组的每个成员的值表示一个程序元素的唯一的运行时可见注解。

  annotation 结构的格式如下：

  ```
  annotation {

  u2 type_index;

  u2 num_element_value_pairs;

  { u2 element_name_index;

  element_value value;

  } element_value_pairs[num_element_value_pairs]

  }
  ```

  annotation 结构各项的说明如下：

  * type_index

    type_index 项的值必须是对常量池的一个有效索引。常量池在该索引处的成员必须是 CONSTANT_Utf8_info（ §4.4.7）结构，表示一个字段描述符， 这个字段描述符表示一个注解类型，和当前 annotation 结构表示的注解一致。

  * num_element_value_pairs

    num_element_value_pairs 项的值给出了当前 annotation 结构表示的注解的键值对（键值对的格式为：元素-值）的数量，即 element_value_pairs[]数组成员数量。需要注意的是，在单独一个注解中可能含有数量最多为 65535个键值对。

  * element_value_pairs[]

    element_value_pairs[]数组的每一个成员的值对应当前 annotation 结构表示的注解中的一个唯一的键值对。 element_value_pairs 的成员包含如下两个项。

    * element_name_index

      element_name_index 项的值必须是对常量池的一个有效索引。常量池在该索引处的成员必须是 CONSTANT_Utf8_info（ §4.4.7）结构，表示一个有效的字段描述符（ §4.3.2），这个字段描述符用于定义当前element_value_pairs 的成员表示的注解的注解名。

    * value

      value 的项的值给出了 element_value_pairs 中的成员的键值对中的值。 


### [element_value 结构](ElementValue.md)