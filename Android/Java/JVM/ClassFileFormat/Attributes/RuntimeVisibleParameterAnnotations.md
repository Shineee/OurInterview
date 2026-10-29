# RuntimeVisibleParameterAnnotations 属性

RuntimeVisibleParameterAnnotations 属性是一个变长属性，位于 method_info （ §4.6）结构的属性表中。用于保存对应方法的参数的所有运行时可见 Java 语言注解。每个 method_info 结构最多只能包含一个 RuntimeVisibleParameterAnnotations 属性，用于保存当前方法的参数的所有可见的 Java 语言注解。 Java 虚拟机必须保证这些注解可被反射的API 使用。

RuntimeVisibleParameterAnnotations 格式如下：

```
RuntimeVisibleParameterAnnotations_attribute {

u2 attribute_name_index;

u4 attribute_length;

u1 num_parameters;

{ u2 num_annotations;

annotation annotations[num_annotations];

} parameter_annotations[num_parameters];

}
```

RuntimeVisibleParameterAnnotations_attribute 结构各项的说明如下：

* attribute_name_index

  attribute_name_index 项的值必须是对常量池的一个有效索引。常量池在该索引处的成员必须是 CONSTANT_Utf8_info（ §4.4.7）结构，表示字符串“ RuntimeVisibleParameterAnnotations”。

* attribute_length

  attribute_length 项的值给出了当前属性的长度，不包括开始的 6 个字节。attribute_length 项的值由对应方法的参数数量，参数的运行时可见注解和它们的值所决定。

* num_parameters

  num_parameters 项的值给出了注解中出现的 method_info（ §4.6）结构表示的方法参数的数量（这些信息可以从方法描述符中获得）。

* parameter_annotations

  parameter_annotations[]数组中每个成员的值表示一个的参数的所有的运行时可见注解。它们的顺序和方法描述符表示的参数的顺序一致。每个parameter_annotations 成员都包含如下两项：

  * num_annotations

    num_annotations 项的值表示 parameter_annotations[]数组在当前所索引处的元素的可见注解的数量。 

  * annotations[]

    annotations[]数组中的每个成员的值表示 parameter_annotations[]数组在当前索引处的元素的一个唯一的可见注解 。