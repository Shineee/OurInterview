# LineNumberTable 属性

LineNumberTable 属性是可选变长属性，位于 Code（ §4.7.3）结构的属性表。它被调试器用于确定源文件中行号表示的内容在 Java 虚拟机的 code[]数组中对应的部分。在 Code 属性的属性表中， LineNumberTable 属性可以按照任意顺序出现，此外，多个 LineNumberTable属性可以共同表示一个行号在源文件中表示的内容，即 LineNumberTable 属性不需要与源文件的行一一对应。
LineNumberTable 属性格式如下：

```
LineNumberTable_attribute {

u2 attribute_name_index;

u4 attribute_length;

u2 line_number_table_length;

{ u2 start_pc;

u2 line_number;

} line_number_table[line_number_table_length];

}
```

LineNumberTable_attribute 结构各项的说明如下：

* attribute_name_index

  attribute_name_index 项的值必须是对常量池的一个有效索引。常量池在该索引处的成员必须是 CONSTANT_Utf8_info（ §4.4.7）结构，表示字符串 “ LineNumberTable”。

* attribute_length

  attribute_length 给出了当前属性的长度，不包括开始的 6 个字节。

* line_number_table_length

  line_number_table_length 项的值给出了 line_number_table[]数组的成员个数。

* line_number_table[]

  line_number_table[]数组的每个成员都表明源文件中行号的变化在 code[]数组中都会有对应的标记点。 line_number_table 的每个成员都具有如下两项：

  * start_pc

    start_pc 项的值必须是 code[]数组的一个索引， code[]数组在该索引处的字符表示源文件中新的行的起点。 start_pc 项的值必须小于当前 LineNumberTable属性所在的 Code 属性的 code_length 项的值。

  * line_number

    line_number 项的值必须与源文件的行数相匹配。 
















