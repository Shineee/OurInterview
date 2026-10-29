# CONSTANT_MethodType_info 结构 

CONSTANT_MethodType_info 结构用于表示方法类型：

```
CONSTANT_MethodType_info {

u1 tag;

u2 descriptor_index;

} 
```

* tag

  CONSTANT_MethodType_info 结构的 tag 项的值为 CONSTANT_MethodType（ 16）。

* descriptor_index

  descriptor_index 项的值必须是对常量池的有效索引， 常量池在该索引处的项必须是CONSTANT_Utf8_info（ §4.4.7）结构，表示方法的描述符（ §4.3.3）。 