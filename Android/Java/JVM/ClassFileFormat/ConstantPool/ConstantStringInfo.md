# CONSTANT_String_info 结构 


CONSTANT_String_info 用于表示 java.lang.String 类型的常量对象，格式如下：

```
CONSTANT_String_info {

u1 tag;

u2 string_index;

}
```

CONSTANT_String_info 结构各项的说明如下：

* tag

  CONSTANT_String_info 结构的 tag 项的值为 CONSTANT_String（ 8）。

* string_index

  string_index 项的值必须是对常量池的有效索引， 常量池在该索引处的项必须是CONSTANT_Utf8_info （ §4.4.7）结构，表示一组 Unicode 码点序列，这组 Unicode码点序列最终会被初始化为一个 String 对象。 