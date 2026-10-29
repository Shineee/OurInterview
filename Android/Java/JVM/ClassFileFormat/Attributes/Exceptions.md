# Exceptions 属性

Exceptions 属性是一个变长属性， 它位于 method_info（ §4.6）结构的属性表中。
Exceptions 属性指出了一个方法需要检查的可能抛出的异常。一个 method_info 结构中最多 只能有一个 Exceptions 属性。

Exceptions 属性格式如下：

```
Exceptions_attribute {

u2 attribute_name_index;

u4 attribute_length;

u2 number_of_exceptions;

u2 exception_index_table[number_of_exceptions];

}
```

Exceptions_attribute 格式的项的说明如下：

* attribute_name_index

  attribute_name_index 项的值必须是对常量池的一个有效索引。常量池在该索引处的成员必须是 CONSTANT_Utf8_info（ §4.4.7）结构，表示字符串"Exceptions"。

* attribute_length

  attribute_length 项的值给出了当前属性的长度，不包括开始的 6 个字节。

* number_of_exceptions

  number_of_exceptions 项的值给出了 exception_index_table[]数组中成员的数量。

* exception_index_table[]

  exception_index_table[]数组的每个成员的值都必须是对常量池的有效索引。 常量池在这些索引处的成员必须都是 CONSTANT_Class_info（ §4.4.1）结构，表示这个方法声明要抛出的异常的类的类型。

一个方法如果要抛出异常，必须至少满足下列三个条件中的一个：

* 要抛出的是 RuntimeException 或其子类的实例。
* 要抛出的是 Error 或其子类的实例。
* 要抛出的是在 exception_index_table[]数组中申明的异常类或其子类的实例。

这些要求没有在 Java 虚拟机中进行强制检查， 它们只在编译时进行强制检查。 




