# CONSTANT_InvokeDynamic_info 结构 

CONSTANT_InvokeDynamic_info 用于表示 invokedynamic 指令所使用到的引导方法（ Bootstrap Method）、 引导方法使用到动态调用名称（ Dynamic Invocation Name）、 参数和请求返回类型、以及可以选择性的附加被称为静态参数（ Static Arguments） 的常量序列。

```
CONSTANT_InvokeDynamic_info {

u1 tag;

u2 bootstrap_method_attr_index;

u2 name_and_type_index;

}
```

CONSTANT_InvokeDynamic_info 结构各项的说明如下：

* tag

  CONSTANT_InvokeDynamic_info 结构的 tag 项的值为

  CONSTANT_InvokeDynamic（ 18）。

* bootstrap_method_attr_index 

  bootstrap_method_attr_index 项的值必须是对当前 Class 文件中引导方法表 （ §4.7.21）的 bootstrap_methods[]数组的有效索引。

* name_and_type_index

  name_and_type_index 项的值必须是对当前常量池的有效索引， 常量池在该索引处的项必须是 CONSTANT_NameAndType_info（ §4.4.6）结构，表示方法名和方法描述符（ §4.3.3）。 




