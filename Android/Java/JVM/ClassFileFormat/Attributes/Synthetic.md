# Synthetic 属性

Synthetic 属性是定长属性，位于 ClassFile（ §4.1）中的属性表。如果一个类成员没有在源文件中出现，则必须标记带有 Synthetic 属性，或者设置 ACC_SYNTHETIC 标志。唯一的例外是某些与人工实现无关的、由编译器自动产生的方法，也就是说， Java 编程语言的默认的实例初始化方法（无参数的实例初始化方法）、类初始化方法，以及 Enum.values()和Enum.valueOf()等方法是不用使用 Synthetic 属性或 ACC_SYNTHETIC 标记的。
Synthetic 属性是在 JDK 1.1 中为了支持内部类或接口而引入的。
Synthetic 属性的格式如下：

```
Synthetic_attribute {

u2 attribute_name_index;

u4 attribute_length;

}
```

Synthetic 结构的说明如下：

* attribute_name_index

  attribute_name_index 项的值必须是对常量池的一个有效索引， 常量池在该索引处的成员必须是 CONSTANT_Utf8_info（ §4.4.7）结构，表示字符串“ Synthetic”。

* attribute_length

  attribute_length 项的值固定为 0。 

















