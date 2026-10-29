# 签名

签名（ Signatures）是用于给 Java 语言使用的描述信息编码，不在 Java 虚拟机系统使用的类型中。泛型类型、方法描述和参数化类型描述等都属于签名。关于签名的详细说明请参考 《 Java语言规范（ Java SE 7 版）》。
Java 编译器需要这类信息来实现（ 或辅助实现） 反射（ reflection） 和跟踪调试功能。
终止符用于表示 Java 编译器产生的各种描述符的名字。 Java 编译器产生的类型有：类型（ Type）、字段（ Field）、局部变量（ Local Variable）、参数（ Parameter）、方法（ Method）、变量类型（ type variable） 等。这些类型的名字中，不能包含字符"."、 ";"、 "["、 "/"、"<"、 ">"和":"，但是可以包含其他 Java 语言没有规定的标识符。
类签名（ Class Signature） 由 ClassSignature 定义， 作用是把 Class 申明的类型信息编译成对应的签名信息。它描述当前类可能包含的所有的（ 泛型类型的） 形式类型参数，如果Class 有直接父类和父接口，类签名会列出它们（ 或将它们参数化）。

```
ClassSignature:

FormalTypeParametersopt SuperclassSignature

SuperinterfaceSignature*
```

一个正式的形式类型参数必须有自己的名字和对这个参数的类或接口的限制（ bounds），限
制跟在类型名之后。如果形式类型参数的类限定没有给定一个具体定义，那么这个参数类型被默认
定义为 java.lang.Object。 

```
FormalTypeParameters:

< FormalTypeParameter+ >

FormalTypeParameter:

Identifier ClassBound InterfaceBound*

ClassBound:

: FieldTypeSignatureopt

InterfaceBound:

: FieldTypeSignature

SuperclassSignature:

ClassTypeSignature

SuperinterfaceSignature:

ClassTypeSignature
```

字段类型签名（ Field Type Signature） 由 FieldTypeSignature 定义， 作用是将字段、参数或局部变量的类型编译成对应的签名信息。

```
FieldTypeSignature:

ClassTypeSignature

ArrayTypeSignature

TypeVariableSignature
```

类的类型签名（ Class Type Signature） 给出了当前类或接口的完整类型信息。类的类型签名必被须明确表达，否则不能准确无误的映射至二进制名，在类的类型签名表示方式中，将内部形式的名称中的符号“ .” 替换成“ $”。

```
ClassTypeSignature:

L PackageSpecifieropt SimpleClassTypeSignature

ClassTypeSignatureSuffix* ;

PackageSpecifier:

Identifier / PackageSpecifier*

SimpleClassTypeSignature:

Identifier TypeArgumentsopt

ClassTypeSignatureSuffix:

. SimpleClassTypeSignature

TypeVariableSignature:

T Identifier ;

TypeArguments:

< TypeArgument+ >

TypeArgument:

WildcardIndicatoropt FieldTypeSignature

*

WildcardIndicator:

+

-

ArrayTypeSignature:

[ TypeSignature

TypeSignature:

FieldTypeSignature

BaseType
```

方法签名（ Method Signature） 由 MethodTypeSignature 定义， 作用是将方法中所有的形式参数的类型编译成相应的签名信息（ 或将它们参数化）。形式参数不包括在 throw 语句中申明的参数类型，返回类型和方法申明的所有的形式类型参数。

```
MethodTypeSignature:

FormalTypeParametersopt (TypeSignature*) ReturnType

ThrowsSignature*

ReturnType:

TypeSignature

VoidDescriptor

ThrowsSignature:

^ ClassTypeSignature

^ TypeVariableSignature
```

如果方法或构造函数不需要抛出异常时，那么 ThowsSignature 就会从MethodTypeSignature 中移除。
如果泛型类，接口，构造函数或成员的泛型签名在 Java 编程语言中包含对类型变量或参数化类型的引用，则 Java 编译器必须这些签名的信息。
对于同一个方法或构造函数，描述符和签名（ §4.3.3） 并不一定完全匹配，这取决于使用的编译器。尤其是 MethodTypeSignature 编译形式参数的 TypeSignatures 数量会少于MethodDescriptor 中 ParameterDescriptors 的数量。
Oracle 发布的 Java 虚拟机实现在加载和链接时，并不校验 Class 文件的签名内容。直到被反射方法调用时才会校验， JDK 的 API 中的 java.lang.reflect 的部分也说明了这点。在未来版本中， Java 虚拟机可能会在启动和链接时对签名加入部分或全部校验。 