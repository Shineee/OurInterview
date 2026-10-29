# CONSTANT_NameAndType_info 结构 

CONSTANT_NameAndType_info 结构用于表示字段或方法，但是和 4.4.2 章节中介绍的 3个结构不同， CONSTANT_NameAndType_info 结构没有标识出它所属的类或接口，格式如下：

```
CONSTANT_NameAndType_info {

u1 tag;

u2 name_index;

u2 descriptor_index;

}
```

CONSTANT_NameAndType_info 结构各项的说明如下：

* tag

  CONSTANT_NameAndType_info 结构的 tag 项的值为 CONSTANT_NameAndType（ 12）。

* name_index

  name_index 项的值必须是对常量池的有效索引， 常量池在该索引处的项必须是CONSTANT_Utf8_info（ §4.4.7）结构，这个结构要么表示特殊的方法名<init>（ §2.9），要么表示一个有效的字段或方法的非限定名（ Unqualified Name）。

* descriptor_index 

  descriptor_index 项的值必须是对常量池的有效索引， 常量池在该索引处的项必须是CONSTANT_Utf8_info（ §4.4.7）结构，这个结构表示一个有效的字段描述符（ §4.3.2）或方法描述符（ §4.3.3）。 