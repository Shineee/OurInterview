# BootstrapMethods 属性

BootstrapMethods 属性是一个变长属性，位于 ClassFile（ §4.1）结构的属性表中。用于保存 invokedynamic 指令引用的引导方法限定符。
如果某个 ClassFile 结构的常量池中有至少一个 CONSTANT_InvokeDynamic_info（ §4.4.10） 项，那么这个 ClassFile 结构的属性表中必须有一个明确的 BootstrapMethods 属性。 ClassFile 结构的属性表中最多只能有一个 BootstrapMethods 属性。
BootstrapMethods 属性格式如下：

```
BootstrapMethods_attribute {

u2 attribute_name_index;

u4 attribute_length;

u2 num_bootstrap_methods;

{ u2 bootstrap_method_ref;

u2 num_bootstrap_arguments;

u2 bootstrap_arguments[num_bootstrap_arguments];

} bootstrap_methods[num_bootstrap_methods];

}
```

BootstrapMethods_attribute 结构各项的说明如下：

* attribute_name_index

  attribute_name_index 项的值必须是对常量池的一个有效索引。常量池在该索引处的成员必须是 CONSTANT_Utf8_info（ §4.4.7）结构，表示字符串“ BootstrapMethods”。

* attribute_length

  ttribute_length 项的值给出了当前属性的长度，不包括开始的 6 个字节。

  attribute_length 项的值由默认值决定①。

* num_bootstrap_methods

  num_bootstrap_methods 项的值给出了 bootstrap_methods[]数组中的引导方法限定符的数量。

* bootstrap_methods[]

  bootstrap_methods[]数组的每个成员包含一个指向 CONSTANT_MethodHandle 结构的索引值，它代表了一个引导方法。还包含了这个引导方法静态参数的序列 （可能为空）。

  bootstrap_methods 每项必须包含以下 3 项：

  * bootstrap_method_ref

    bootstrap_method_ref 项的值必须是一个对常量池的有效索引。常量池在该索引处的值必须是一个 CONSTANT_MethodHandle_info 结构。注意： 此 CONSTANT_MethodHandle_info 结构的 reference_kind 项应为值 6（ REF_invokeStatic）或 8（ REF_newInvokeSpecial）（ §5.4.3.5），否则在 invokedynamic 指令解析调用点限定符时，引导方法会执行失败。

  * num_bootstrap_arguments

    num_bootstrap_arguments 项的值给出了 bootstrap_arguments[]数组成员的数量。

  * bootstrap_arguments[]

    bootstrap_arguments[]数组的每个成员必须是一个对常量池的有效索引。 常量池在该索引出必须是下列结构之一：

    CONSTANT_String_info,CONSTANT_Class_info、

    CONSTANT_Integer_info,CONSTANT_Long_info、 

    CONSTANT_Float_info,CONSTANT_Double_info、

    CONSTANT_MethodHandle_info 或 CONSTANT_MethodType_info。 