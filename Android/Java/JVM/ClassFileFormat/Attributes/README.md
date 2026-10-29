# 属性

属性（ Attributes）在 Class 文件格式中的 ClassFile（ §4.1）结构、 field_info （ §4.5）结构， method_info（ §4.6）结构和 Code_attribute（ §4.7.3）结构都有使用，所
有属性的通用格式如下：

```
attribute_info {

u2 attribute_name_index;

u4 attribute_length;

u1 info[attribute_length];

}
```

对于任意属性， attribute_name_index 必须是对当前 Class 文件的常量池的有效 16 位无符号索引。 常量池在该索引处的项必须是 CONSTANT_Utf8_info（ §4.4.7）结构，表示当前属性的名字。 attribute_length 项的值给出了跟随其后的字节的长度，这个长度不包括attribute_name_index 和 attribute_name_index 项的 6 个字节。
有些属性因 Class 文件格式规范所需，已被预先定义好。这些属性在表 4.6 中列出，同时，被列出的信息还包括它们首次出现的 Class 文件版本和 Java SE 版本号。在本规范定义的环境中， 也就是已包含这些预定义属性的 Class 文件中， 它们的属性名称被保留，不能再被属性表中其他的自定义属性所使用。
表 4.6 被预定义的 Class 文件属性 

|                 属性名                  | Java SE | Class 文件 |
| :----------------------------------: | :-----: | :------: |
|            ConstantValue             |  1.0.2  |   45.3   |
|                 Code                 |  1.0.2  |   45.3   |
|            StackMapTable             |    6    |   50.0   |
|              Exceptions              |  1.0.2  |   45.3   |
|             InnerClasses             |   1.1   |   45.3   |
|           EnclosingMethod            |   5.0   |   49.0   |
|              Synthetic               |   1.1   |   45.3   |
|              Signature               |   5.0   |   49.0   |
|              SourceFile              |  1.0.2  |   45.3   |
|         SourceDebugExtension         |   5.0   |   49.0   |
|           LineNumberTable            |  1.0.2  |   45.3   |
|          LocalVariableTable          |  1.0.2  |   45.3   |
|        LocalVariableTypeTable        |   5.0   |   49.0   |
|              Deprecated              |   1.1   |   45.3   |
|      RuntimeVisibleAnnotations       |   5.0   |   49.0   |
|     RuntimeInvisibleAnnotations      |   5.0   |   49.0   |
|  RuntimeVisibleParameterAnnotations  |   5.0   |   49.0   |
| RuntimeInvisibleParameterAnnotations |   5.0   |   49.0   |
|          AnnotationDefault           |   5.0   |   49.0   |
|           BootstrapMethods           |    7    |   51.0   |

* Java 虚拟机实现的 Class 文件加载器（ Class File Reader） 必须正确的识别和读取 ConstantValue， Code 和 Exceptions 属性；同样， Java 虚拟机也必须能正确的解析它们的语义。
* InnerClasses， EnclosingMethod 和 Synthetic 属性必须被 Class 文件加载器正确的识别并读入， 它们用于实现 Java 平台的类库（ §2.12）。
* 如果 Java 虚拟机实现支持的 Class 文件的版本号为 49.0 或更高时，它的 Class 文件加载器必须能正确的识别并读取 Class 文件中的 RuntimeVisibleAnnotations，RuntimeInvisibleAnnotations， RuntimeVisibleParameterAnnotations，RuntimeInvisibleParameterAnnotations 和 AnnotationDefault 属性， 它们用于实现 Java 平台类库（ §2.12）。 
* 如果 Java 虚拟机实现支持的 Class 文件的版本号为 49.0 或更高时，它的 Class 文件加载器必须正确的识别和读取 Class 文件中的 Signature 属性。
* 如果 Java 虚拟机实现支持的 Class 文件的版本号为 50.0 或更高时，它的 Class 文件加载器必须正确的识别和读取 StackMapTable 属性。
* 如果 Java 虚拟机实现支持的 Class 文件的版本号为 51.0 或更高时，它的 Class 文件加载器必须正确的识别和读取 BootstrapMethods 属性。
* 对于剩余的预定义属性的使用不受限制；如果剩余的预定义属性包含虚拟机可识别的信息， Class 文件加载器就可以选择使用这些信息，否则可以选择忽略它们。 


### [自定义和命名新的属性](CustomizeAndNameNewProperties.md)

### [ConstantValue属性](ConstantValue.md)

### [Code属性](Code.md)

### [StackMapTable属性](StackMapTable.md)

### [Exceptions属性](Exceptions.md)

### [InnerClasses属性](InnerClasses.md)

### [EnclosingMethod属性](EnclosingMethod.md)

### [Synthetic属性](Synthetic.md)

### [Signature属性](Signature.md)

### [SourceFile属性](SourceFile.md)

### [SourceDebugExtension属性](SourceDebugExtension.md)

### [LineNumberTable 属性](LineNumberTable.md)

### [LocalVariableTable 属性](LocalVariableTable.md)

### [LocalVariableTypeTable 属性](LocalVariableTypeTable.md)

### [Deprecated 属性](Deprecated.md)

### [RuntimeInvisibleAnnotations 属性](RuntimeInvisibleAnnotations.md)

### [RuntimeVisibleParameterAnnotations 属性](RuntimeVisibleParameterAnnotations.md)

### [RuntimeInvisibleParameterAnnotations 属性](RuntimeInvisibleParameterAnnotations.md)

### [AnnotationDefault 属性](AnnotationDefault.md)

### [BootstrapMethods 属性](BootstrapMethods.md)