# 语法符号


描述符和签名都是用特定的语法符号（ Grammar）来表示，这些语法是一组可表达如何根据不同的类型去产生可恰当描述它们的字符序列的标识集合。在本规范中， 语法的终止符号用定长的黑体字表示。非终止符号用斜体字表示，非终止符的定义由被定义的非终止名后跟随一个冒号表示。冒号右侧的一个或多个可交换的非终止符连续排列，每个非终止符占一行①。 例如：

```java
FieldType:

BaseType

ObjectType

ArrayType
```

上面文字表达的意思是 FileType 可以表示 BaseType、 ObjectType、 ArrayType 三者之一。
当一个有星号（ *） 跟随的非终止符出现在一个语法标识的右侧时，说明带有这个非终止符的语法标识将产生 0 或多个不同值，这些值按照顺序且无间隔的排列在非终止符后面。当一个有加号（ +） 跟随的非终止符出现在一个结构的右侧时，说明这个非终止符将产生一个或多个不同值，这些值按照顺序且无间隔的排列在非终止符后面。 例如：

```java
MethodDescriptor:

( ParameterDescriptor* ) ReturnDescriptor
```

上面文字表达的意思是 MethodDescriptor 的是由左括号“ (”、 0 或若干个连续排列的ParameterDescriptor 值、 右括号“ )”、 ReturnDescriptor 值构成。 








