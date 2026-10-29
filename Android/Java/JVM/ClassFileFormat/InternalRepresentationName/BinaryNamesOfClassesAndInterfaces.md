# 类和接口的二进制名称

在 Class 文件结构中出现的类或接口的名称， 都通过全限定形式（ Fully Qualified Form）来表示， 这被称作它们的“二进制名称”。这个名称使用 CONSTANT_Utf8_info结构来表示，因此如果忽略其他一些约束限制的话，这个名称可能来自整个 Unitcode字符空间的任意字符组成。 类和接口的二进制名称还会被 CONSTANT_NameAndType_info结构所引用，用于构成它们的描述符，引用这些名称通过引用它们的CONSTANT_Class_info 结构来实现。
由于历史原因，出现在 Class 文件结构中的二进制名称的语法与《 Java 语言规范》中§13.1规定的语法二进制名格式有差别。在本规范规定的内部形式中， 用来分隔各个标识符的符号不在是ASCII 字符点号（ '.'），而是被 ASCII 字符斜杠（ '/'）所代替，每个标识符都是一个非全限定名（ Unqualified Names， §4.2.2） ①。
譬如，类 Thread 的正常的二进制名是 java.lang.Thread。在 Class 文件的内部表示形式里面，对类 java.lang.Thrad 的引用是通过来一个代表字符串“ java/lang/Thread” 的CONSTANT_Utf8_info 结构来实现的。 



