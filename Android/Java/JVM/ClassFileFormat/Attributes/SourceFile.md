# SourceFile 属性

SourceFile 属性是可选定长字段，位于 ClassFile（ §4.1）结构的属性表。一个ClassFile 结构中的属性表最多只能包含一个 SourceFile 属性。 

SourceFile 属性格式如下：

```
SourceFile_attribute {

u2 attribute_name_index;

u4 attribute_length;

u2 sourcefile_index;

}
```

SourceFile_attribute 结构各项的说明如下：

* attribute_name_index

  attribute_name_index 项的值必须是一个对常量池的有效索引。 常量池在该索引处的成员必须是 CONSTANT_Utf8_info（ §4.4.7）结构，表示字符串“ SourceFile”。

* sourcefile_index

  sourcefile_index 项的值必须是一个对常量池的有效索引。 常量池在该索引处的成员必须是 CONSTANT_Utf8_info（ §4.4.7）结构，表示一个字符串。

  sourcefile_index 项引用字符串表示被编译的 Class 文件的源文件的名字。不包括源文件所在目录的目录名，也不包括源文件的绝对路径名。平台相关（绝对路径名等） 的附加信息必须是运行时解释器（ Runtime Interpreter）或开发工具在文件名实际使用时提供。 













