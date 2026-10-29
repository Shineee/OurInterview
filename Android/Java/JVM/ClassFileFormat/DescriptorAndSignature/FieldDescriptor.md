# 字段描述符


字段描述符（ Field Descriptor），是一个表示类、 实例或局部变量的语法符号，是由语法产生的字符序列：

```java
FieldDescriptor:

FieldType

ComponentType:

FieldType

FieldType:

BaseType

ObjectType

ArrayType

BaseType:

B C D F I J S Z

ObjectType:

L Classname ;

ArrayType:

[ ComponentType
```

所有表示基本类型（ BaseType） 的字符、表示对象类型（ ObjectType） 中的字符"L"， 表示数组类型（ ArrayType） 的"["字符都是 ASCII 编码的字符。对象类型（ ObjectType） 中的Classname 表示一个类或接口二进制名称的内部格式（ §4.2.1）。表示数组类型的有效描述符的长度应小于等于 255。所有字符类型的解释如表 4.2 所示。

表 4.2 基本类型字符解释表 

|      字符      |    类型     |         含义         |
| :----------: | :-------: | :----------------: |
|      B       |   byte    |      有符号字节型数       |
|      C       |   char    | Unicode字符，UTF-16编码 |
|      D       |  double   |       双精度浮点数       |
|      F       |   float   |       单精度浮点数       |
|      I       |    int    |        整型数         |
|      J       |   long    |        长整数         |
|      S       |   short   |       有符号短整数       |
|      Z       |  boolean  |   布尔值 true/false   |
| L Classname; | reference | 一个名为<Classname>的实例 |
|      [       | reference |       一个一维数组       |


举个例子：描述 int 实例变量的描述符是“I”； java.lang.Object 的实例描述符是 “Ljava/lang/Object;”。注意，这里用到了类 Object 的二进制名的内部形式（此处是内部形式）。 double 的三维数组“ double d[][][];” 的描述符为“ [[[D”。 
