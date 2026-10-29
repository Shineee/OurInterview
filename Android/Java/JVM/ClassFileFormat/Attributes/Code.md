# Code 属性

Code 属性是一个变长属性，位于 method_info（ §4.6）结构的属性表。一个 Code 属性只为唯一一个方法、实例类初始化方法或类初始化方法（ §2.9）保存 Java 虚拟机指令及相关辅助信息。 所有 Java 虚拟机实现都必须能够识别 Code 属性。如果方法被声明为 native 或者abstract 类型，那么对应的 method_info 结构不能有明确的 Code 属性，其它情况下，method_info 有必须有明确的 Code 属性。
Code 属性的格式如下：

```
Code_attribute {

u2 attribute_name_index;

u4 attribute_length;

u2 max_stack;

u2 max_locals;

u4 code_length;

u1 code[code_length];

u2 exception_table_length;

{ u2 start_pc;

u2 end_pc;

u2 handler_pc;

u2 catch_type;

} exception_table[exception_table_length];

u2 attributes_count;

attribute_info attributes[attributes_count];

}
```

Code_attribute 结构各项的说明如下：

* attribute_name_index

  attribute_name_index 项的值必须是对常量池的有效索引， 常量池在该索引处的项必须是 CONSTANT_Utf8_info（ §4.4.7）结构，表示字符串“ Code”。 



* attribute_length

  attribute_length 项的值表示当前属性的长度，不包括开始的 6 个字节。

* max_stack

  max_stack 项的值给出了当前方法的操作数栈在运行执行的任何时间点的最大深度（ §2.6.2）。

* max_locals

  max_locals 项的值给出了分配在当前方法引用的局部变量表中的局部变量个数，包括调用此方法时用于传递参数的局部变量。

  long 和 double 型的局部变量的最大索引是 max_locals-2，其它类型的局部变量的最大索引是 max_locals-1.

* code_length

  code_length 项给出了当前方法的 code[]数组的字节数①， code_length 的值必须大于 0，即 code[]数组不能为空。

* code[]

  code[]数组给出了实现当前方法的 Java 虚拟机字节码。

  code[]数组以按字节寻址的方式读入机器内存，如果 code[]数组的第一个字节是按以4 字节边界对齐的话，那么 tableswitch 和 lookupswitch 指令中所有涉及到的 32位偏移量也都是按 4 字节长度对齐的（ 关于 code[]数组边界对齐对字节码的影响， 请参考相关的指令描述）。

  本规范对关于 code[]数组内容的详细约束有很多， 将在后面单独章节（ §4.9） 中列出。

* exception_table_length

  exception_table_length 项的值给出了 exception_table[]数组的成员个数量。

* exception_table[]

  exception_table[]数组的每个成员表示 code[]数组中的一个异常处理器（ Exception Handler）。 exception_table[]数组中， 异常处理器顺序是有意义的（不能随意更改），详细内容见 2.10 节。

  exception_table[]数组包含如下 4 项： 

  * start_pc 和 end_pc

    start_pc 和 end_pc 两项的值表明了异常处理器在 code[]数组中的有效范围。start_pc 必须是对当前 code[]数组中某一指令的操作码的有效索引， end_pc 要么是对当前 code[]数组中某一指令的操作码的有效索引，要么等于 code_length的值，即当前 code[]数组的长度。 start_pc 的值必须比 end_pc 小。当程序计数器在范围[start_pc, end_pc)内时， 异常处理器就将生效。即设 x 为异常句柄的有效范围内的值， x 满足： start_pc ≤ x < end_pc。实际上， end_pc 值本身不属于异常处理器的有效范围这点属于 Java 虚拟机历史上的一个设计缺陷：如果 Java 虚拟机中的一个方法的 code 属性的长度刚好是 65535个字节，并且以一个 1 个字节长度的指令结束，那么这条指令将不能被异常处理器所处理。不过编译器可以通过限制任何方法、 实例初始化方法或类初始化方法的code[]数组最大长度为 65534， 这样可以间接弥补这个 BUG。

  * handler_pc

    handler_pc 项表示一个异常处理器的起点，它的值必须同时是一个对当前 code[]数组中某一指令的操作码的有效索引。

  * catch_type

    如果 catch_type 项的值不为 0，那么它必须是对常量池的一个有效索引， 常量池在该索引处的项必须是 CONSTANT_Class_info（ §4.4.1）结构，表示当前异常处理器指定需要捕捉的异常类型。只有当抛出的异常是指定的类或其子类的实例时，异常处理器才会被调用。

    如果 catch_type 项的值如果为 0，那么这个异常处理器将会在所有异常抛出时都被调用。这可以用于实现 finally 语句（ §3.13，“ 编译 finally”）。

* attributes_count

  attributes_count 项的值给出了 Code 属性中 attributes 表的成员个数。

* attributes[]

  属性表的每个成员的值必须是 attribute 结构（ §4.7）。一个 Code 属性可以有任意数量的可选属性与之关联。

  本规范中定义的、 可以出现在 Code 属性的属性表中的成员只能是 LineNumberTable（ §4.7.12）， LocalVariableTable（ §4.7.13）， LocalVariableTypeTable （ §4.7.14）和 StackMapTable（ §4.7.4）属性。

  如果一个 Java 虚拟机实现支持的 Class 文件版本号为 50.0 或更高，那么它必须正确的识别和读取 Code 属性的属性表出现的 StackMapTable（ §4.7.4）属性。Java 虚拟机实现必须自动忽略 Code 属性的属性表数组中出现的所有它不能识别属性。本规范中没有定义的属性不可影响 Class 文件的语义，只能提供附加描述信息（ §4.7.1）。 




