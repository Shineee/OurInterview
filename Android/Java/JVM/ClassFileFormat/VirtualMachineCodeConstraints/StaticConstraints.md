# 静态约束 

Class 文件的静态约束（ Static Constraints）是一系列对文件是否编排良好的定义。在之前章节描述到 Class 文件中的 Java 虚拟机代码静态约束而导致的异常时，就已经提及过这些约束。Java 虚拟机代码对 Class 文件中的静态约束确定了 Java 虚拟机指令在 code[]数组中是如何排列的，某些特殊的指令必须带有哪些操作数等等。
code[]数组中的指令的约束情况如下：

1. code[]数组不能为空，因此 code_length 项的值也一定大于 0。
2. code_length 项的值必须小于 65536。 


3. code[]数组中第一条指令的操作码是从数组中索引为 0 处开始。

4. code[]数组中仅仅只能出现 6.4 章节中所列的指令。那些被保留使用的或没有列在本规范的操作码的指令一定不允许出现在 code[]数组中。

5. 如果 Class 文件的版本号大于或等于 51.0， jsr 和 jsr_w 这两个操作码也不能出现在code[]数组中。

6. 对于 code[]数组中除最后一条指令外的其它指令来说，后一条指令的操作码的索引等于当前指令操作码的索引加上当前指令的长度（包含指令带有的操作数）。 wide 指令在这种情况下将与其它的指令使用相同的规则进行处理：跟随在 wide 指令之后的操作码将会被视为 wide 指令的操作数。程序跳转时不能被直接跳转到该操作码。

7. code[]数组中最后一条指令的最后一个字节的索引必须等于 code_length 值减 1。

   code[]数组中的指令的操作数的约束情况如下：

   * 所有跳转和分支指令（ jsr、 jsr_w、 goto、 goto_w、 ifeq、 ifne、 ifle、 iflt、ifgt、 ifnull、 ifnonnull、 if_icmpeq、 if_icmpne、 if_icmple、 if_icmplt、if_icmpge、 if_icmpgt、 if_acmpeq、 if_acmpne）的跳转目标必须是本方法内某条指令的操作码。但跳转与分支指令的目标一定不能是某条被 wide 指令修饰的指令的操作码；跳转与分支指令的跳转目标可以是 wide 指令本身。
   * 每个 tableswitch 指令的跳转目标（包含 default 目标）都必须是当前方法内的某条指令的操作码。每个 tableswitch 指令必须在它的跳转表中包含与自己 low 和 high跳转表操作数的值相等的条目，并且 low 值必须小于等于 high 值。 tableswitch 指令的跳转目标也不能是某条被 wide 指令修饰的指令的操作码； tableswitch 的跳转目标可能是 wide 指令本身。
   * 每个 lookupswitch 指令的跳转目标（包含 default 目标）都必须是当前方法内的某条指令的操作码。每个 lookupswitch 指令都必须包含与自己 npairs 操作数的值相等数量的 match-offset 值对。 match-offset 值对必须是以 match 值递增的顺序排列的。 lookupswitch 指令的跳转目标也不能是某条被 wide 指令修饰的指令的操作码；lookupswitch 指令的跳转目标可能是 wide 指令本身。
   * 每个 ldc 和 ldc_w 指令的操作数都必须是 constant_pool 表中的一个有效索引。操作数必须表示到 constant_pool 表的有效索引。被此索引所引用的常量池项必须符合下列规则： 
     - 如果 Class 文件版本号小于 49.0，那就可能是 CONSTANT_Integer，CONSTANT_Float 或 CONSTANT_String。
     - 如果 Class 文件版本号是 49.0 或 50.0，那就可能是 CONSTANT_Integer，CONSTANT_Float， CONSTANT_String 或 CONSTANT_Class。
     - 如果 Class 文件版本号是 51.0，那就可能是 CONSTANT_Integer，CONSTANT_Float， CONSTANT_String， CONSTANT_Class，CONSTANT_MethodType 或 CONSTANT_MethodHandle。
   * 每个 ldc2_w 指令的操作数都必须是 constant_pool 表内的一个有效索引。被此索引所引用的常量池项必须是 CONSTANT_Long 或 CONSTANT_Double 之一。并且在紧随该常量池索引之后那个常量池项一定不能被独立使用。
   * 每个 getfield、 putfield、 getstatic 和 putstatic 指令的操作数都必须是constant_pool 表内的一个有效索引。由这些索引所引用的常量池项必须是CONSTANT_Fieldref 类型。
   * 每个 invokevirtual、 invokespecial 和 invokestatic 指令的索引位操作数（ Indexbyte Operand）都必须是 constant_pool 表内的有效索引。由这些索引所引用的常量池项必须是 CONSTANT_Methodref 类型。
   * 每个 invokedynamic 指令的索引位操作数必须表示对 constant_pool 表内的有效索引。由此索引所引用的常量池项必须是 CONSTANT_InvokeDynamic 类型。所有invokedynamic 指令的第三和第四个操作数字节必须是 0。
   * 只有 invokespecial 指令可以调用实例初始化方法（ §2.9）。其他所有以’<’（ ’\u003c’）字符开头的方法都不能再被方法调用指令所调用。需要特别说明的是，被命名为<clinit>的类和接口的初始化方法也不会被 Java 虚拟机指令所调用，而是由Java 虚拟机自己来隐式调用。
   * 每个 invokeinterface 指令的索引位操作数都必须表示对 constant_pool 表的有效索引。被此索引所引用的常量池项必须是 CONSTANT_InterfaceMethodref 类型。每个invokeinterface指令的count操作数的值必须真实反映出可传入到接口方法的参数所占用的局部变量个数，它由 CONSTANT_InteraceMethodref 项所引用的CONSTANT_NameAndType_info 结构的描述符所暗示。所有 invokeinterface 指令的第四个操作数字节必须是 0。 
   * instanceof， checkcast， new 和 anewarray 指令的操作数，和 multianewarray指令的索引位操作数都必须表示对 constant_pool 表的有效索引。由这些索引所引用的常量池项必须是 CONSTANT_Class 类型。
   * anewarray 指令不能用于创建维度超过 255 维的数组。
   * new 指令引用的 contant_pool 表中 CONSTANT_Class 项来表示创建的对象类型，这个类型不能是数组类型。也就是说 new 指令不能用于创建数组。
   * multianewarray 指令只能创建一个最多不超过它 dimensions 操作数值所代表的维度的数组。那就是说， multianewarray 指令创建的数组的维度可以比操作数所示的小（注：多维数组中某一维度的数组长度为 0 的情况），但是它所创建的数组维度绝对不能比操作数中所示的维度值要多。 multianewarray 指令的 dimensions 操作数必须不能为 0。
   * 每个 newarray 指令的 atype 操作数必须为下列类型的值： T_BOOLEAN（ 4）， T_CHAR（ 5）， T_FLOAT（ 6）， T_DOUBLE（ 7）， T_BYTE（ 8）， T_SHORT（ 9）， T_INT（ 10）和 T_LONG（ 11）。
   * 所有 iload， fload， aload， istore， fstore， astore， iinc 和 ret 指令的索引位操作数必须是非负整数且不能大于值 max_locals-1。
   * 所有 iload_<n>， fload_<n>， aload_<n>， istore_<n>， fstore_<n>和astore_<n>指令的显式索引必须小于等于值 max_locals–1。
   * 所有 lload， dload， lstore 和 dstore 指令的索引位操作数必须小于等于值max_locals–2。
   * 所有 lload_<n>， dload_<n>， lstore_<n>和 dstore_<n>指令的显式索引必须小于或等于值 max_locals–2。
   * 修饰 iload， fload， aload， istore， fstore， astore， ret 或 innc 指令的 wide指令的索引位操作数必须表示非负整数且小于等于值 max_locals–1。修改 lload，dload， lstore 和 dstore 指令的 wide 指令的索引位操作数必须表示非负整数且小于或等于值 max_locals–2。 

