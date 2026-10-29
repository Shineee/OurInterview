# element_value 结构 

element_value 结构是一个可辨识联合体（ Discriminated Union） ①，用于表示“元素-值”的键值对中的值。它被用来描述所有注解（包括 RuntimeVisibleAnnotations、RuntimeInvisibleAnnotations、 RuntimeVisibleParameterAnnotations 和untimeInvisibleParameterAnnotations）中涉及到的元素值。
element_value 的结构格式如下：

```
element_value {

u1 tag; 

union {

u2 const_value_index;

{ u2 type_name_index;

u2 const_name_index;

} enum_const_value;

u2 class_info_index;

annotation annotation_value;

{ u2 num_values;

element_value values[num_values];

} array_value;

} value;

}
```

element_value 结构各项的说明如下。

* tag

  tag 项表明了当前注解的元素-值对的类型。 tag 值为字母'B'、 'C'、 'D'、 'F'、 'I'、'J'、 'S'和'Z'时表示的含义和章节 4.3.2 中表 4.2 所定义的一样。其余 tag 的预定义值和对应解释在表 4.9 列出。

  表 4.9 附加的 tag 值解释 

  | tag值 |      元素类型       |
  | :--: | :-------------: |
  |  s   |     String      |
  |  e   |  enum contant   |
  |  c   |      class      |
  |  @   | annotation type |
  |  [   |      array      |

* value

  value 项的表示当前注解元素的值。 此项是一个联合体结构， 当前项的 tag 项决定了这个联合体结构的哪一项会被使用：

  * type_name_index

    type_name_index 项的值必须是对常量池的一个有效索引。常量池在该索引处的成员必须是 CONSTANT_Utf8_info（ §4.4.7）结构，表示一个有效的字段描述符（ §4.3.2），这个字段描述符表示当前 element_value 结构所表示的枚举常量类型的内部形式的二进制名称（ § 4.2.1）。

  * const_name_index

    const_name_index 项的值必须是对常量池的一个有效索引。常量池在该索引处的成员必须是 CONSTANT_Utf8_info（ §4.4.7）结构，表示一个有效的字段描述符（ §4.3.2），这个字段描述符表示当前 element_value 结构所表示的枚举常量类型的简单名称。

  * class_info_index

    当 tag 项为'c'时， class_info_index 项才会被使用。 class_info_index 项的值必须是对常量池的一个有效索引。常量池在该索引处的成员必须是CONSTANT_Utf8_info（ §4.4.7）结构，表示返回描述符（ §4.3.3）的类型，这个类型由当前 element_value 结构所表示的类型决定（ 譬如： 'V'表示 Void，'Ljava/lang/Object;'表示类 java.lang.Object 等）。

  * annotation_value

    当 tag 项为'@'时， annotation_value 项才会被使用。这时 element_value结构表示一个内部的注解（ Nested Annotation）。

  * array_value

    当 tag 项为'['时， array_value 项才会被使用。array_value 项包含如下两项：

    * num_values

      num_values 项的值给定了当前 element_value 结构表示的数组类型的成员的数量。注意，允许数组类型元素中最多有 65535 个元素。

    * values

      values 的每个成员的值都给指明了当前 element_value 结构所表示的数组类型的一个元素值。 














