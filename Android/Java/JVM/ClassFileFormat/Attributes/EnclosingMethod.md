# EnclosingMethod 属性

EnclosingMethod 属性是可选的定长属性，位于 ClassFile（ §4.1）结构的属性表。当且仅当 Class 为局部类或者匿名类时，才能具有 EnclosingMethod 属性。一个类最多只能有一个 EnclosingMethod 属性。
EnclosingMethod 属性格式如下：

```
EnclosingMethod_attribute {

u2 attribute_name_index;

u4 attribute_length;

u2 class_index

u2 method_index;

}
```

EnclosingMethod_attribute 结构的项说明如下：

* attribute_name_index

  attribute_name_index 项的值必须是一个对常量池的有效索引。 常量池在该索引处的项必须是 CONSTANT_Utf8_info（ §4.4.7）结构，表示字符串"EnclosingMethod"。

* attribute_length

  attribute_length 项的值固定为 4。

* class_index

  class_index 项的值必须是一个对常量池的有效索引。 常量池在该索引出的项必须是CONSTANT_Class_info（ §4.4.1）结构，表示包含当前类声明的最内层类。

* method_index 

  如果当前类不是在某个方法或初始化方法中直接包含（ Enclosed），那么method_index 值为 0，否则 method_index 项的值必须是对常量池的一个有效索引，常量池在该索引处的成员必须是 CONSTANT_NameAndType_info（ §4.4.6）结构，表示由 class_index 属性引用的类的对应方法的方法名和方法类型。 Java 编译器有责任在语法上保证通过 method_index 确定的方法是语法上最接近那个包含EnclosingMethod 属性的类的方法（ Closest Lexically Enclosing Method）。 














