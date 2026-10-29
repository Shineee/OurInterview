# 常量池

Java 虚拟机指令执行时不依赖与类、接口，实例或数组的运行时布局，而是依赖常量池
（ constant_pool） 表中的符号信息。 

所有的常量池项都具有如下通用格式：

```
cp_info {

u1 tag;

u1 info[];

}
```

常量池中，每个 cp_info 项的格式必须相同， 它们都以一个表示 cp_info 类型的单字节“ tag” 项开头。 后面 info[]项的内容 tag 由的类型所决定。 tag 有效的类型和对应的取值在表4.3 列出。每个 tag 项必须跟随 2 个或更多的字节，这些字节用于给定这个常量的信息，附加字节的信息格式由 tag 的值决定。 

表 4.3 常量池的 tag 项说明

| 常量类型                        | 值    |
| --------------------------- | ---- |
| CONSTANT_Class              | 7    |
| CONSTANT_Fieldref           | 9    |
| CONSTANT_Methodref          | 10   |
| CONSTANT_InterfaceMethodref | 11   |
| CONSTANT_String             | 8    |
| CONSTANT_Integer            | 3    |
| CONSTANT_Float              | 4    |
| CONSTANT_Long               | 5    |
| CONSTANT_Double             | 6    |
| CONSTANT_NameAndType        | 12   |
| CONSTANT_Utf8               | 1    |
| CONSTANT_MethodHandle       | 15   |
| CONSTANT_MethodType         | 16   |
| CONSTANT_InvokeDynamic      | 18   |

### [CONSTANT_Class_info 结构](ConstantClassInfo.md)

### [CONSTANT_Fieldref_info， CONSTANT_Methodref_info 和CONSTANT_InterfaceMethodref_info 结构 ](ConstantFieldrefMethodrefInterfaceMethodrefInfo.md)

### [CONSTANT_String_info 结构](ConstantStringInfo.md)

### [CONSTANT_Integer_info 和 CONSTANT_Float_info 结构](ConstantIntegerFloatInfo.md)

### [CONSTANT_Long_info 和 CONSTANT_Double_info 结构](ConstantLongDoubleInfo.md)

### [CONSTANT_NameAndType_info 结构](ConstantNameAndTypeInfo.md)

### [CONSTANT_Utf8_info 结构](ConstantUtf8Info.md)

### [CONSTANT_MethodHandle_info 结构](ConstantMethodHandleInfo.md)

### [CONSTANT_MethodType_info 结构](ConstantMethodTypeInfo.md)

### [CONSTANT_InvokeDynamic_info 结构](ConstantInvokeDynamicInfo.md)