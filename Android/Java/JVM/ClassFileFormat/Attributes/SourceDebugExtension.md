# SourceDebugExtension 属性

SourceDebugExtension 属性是可选属性，位于 ClassFile（ §4.1）结构属性表。ClassFile 结构的属性表最多只能包含一个 SourceDebugExtension 属性。
SourceDebugExtension 属性的格式如下：

```
SourceDebugExtension_attribute {

u2 attribute_name_index;

u4 attribute_length;

u1 debug_extension[attribute_length];

}
```

SourceDebugExtension_attribute 结构各项的说明如下：

* attribute_name_index

  attribute_name_index 项的值必须是对常量池的一个有效索引。常量池在该索引处的成员必须是 CONSTANT_Utf8_info（ §4.4.7）结构，表示字符串“ SourceDebugExtension”。

* attribute_length

  attribute_length 项的值给出了当前属性的长度，不包括开始的 6 个字节。即attribute_length 项的值是字节数组 debug_extension[]数组的长度

* debug_extension[]

  debug_extension[]数组用于保存扩展调试信息，扩展调试信息对于 Java 虚拟机来说没有实际的语义。这个信息用改进版的 UTF-8 编码的字符串（ §4.4.7）表示，这个字符串不包含 byte 值为 0 的终止符。需要注意的是， debug_extension[]数组表示的字符串可以比 Class 实例对应的字符串更长。 












