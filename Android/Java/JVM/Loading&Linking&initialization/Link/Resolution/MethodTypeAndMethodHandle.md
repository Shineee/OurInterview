# 方法类型与方法句型解析

为了解析方法类型（ Method Type）的未解析符号引用， 此方法类型所封装的方法描述符中所有的类型符号引用（ Symbolic References To Classes） 都必须先被解析（ §5.4.3.1）。因此，在解析这些类型符号引用的过程中如果有任何异常发生，也会当作解析方法类型的异常而被抛出。
解析方法类型的结果是得到一个对 java.lang.invoke.MethodType 实例的引用，它可用来表示一个方法的描述符。
解析方法句柄的符号引用的过程会更为复杂。每个被 Java 虚拟机解析的方法句柄都有一个被称为字节码行为（ Bytecode Behavior） 的等效指令序列（ Equivalent InstructionSequence），它由方法句柄的类型（ Kind） 值来标识。九种方法句柄的类型值和描述见下表所示。
指令序列到字段或方法的符号引用被标记为： C.x:T。这里的 x 和 T 分别表示字段和方法的名称和描述符（ §4.3.2、 §4.3.3）， C 表示字段或方法所属的类或接口。 

|  类型  |          描述          |                  字节码行为                   |
| :--: | :------------------: | :--------------------------------------: |
|  1   |     REF_getField     |              getfield C.f:T              |
|      |    REF_getStatic     |             getstatic C.f:T              |
|      |     REF_putField     |              putfield C.f:T              |
|      |    REF_putStatic     |             putstatic C.f:T              |
|      |  REF_invokeVirtual   |         invokevirtual C.m:(A*) T         |
|      |   REF_invokeStatic   |         invokestatic C.m:(A*) T          |
|      |  REF_invokeSpecial   |         invokespeical C.m:(A*) T         |
|      | REF_newInvokeSpecial | new C;dup;invokespecial C.<init>:(A*)void |
|      | REF_invokeInterface  |        invokeinterface C.m:(A*) T        |

假设 MH 表示正在被解析的方法句柄（ §5.1）的符号引用，那么：

* 设 R 是 MH 中字段或方法的符号引用。

（ MH 是从 CONSTANT_MethodHandle 结构中得来，它的 reference_index 项的索引所指向的 CONSTANT_Fieldref， CONSTANT_Methodref 或CONSTANT_InterfaceMethodref 结构可以得到 R。）

* 设 C 是 R 所引用的类型的符号引用。

（ C 是由表示 R 的 CONSTANT_Fieldref， CONSTANT_Methodref 和CONSTANT_InterfaceMethodref 处 class_index 项引用的 CONSTANT_Class 结构所得到。）

* 设 f 或 m 是 R 所引用的字段或方法的名称。

（ f 或 m 是由 R 的 CONSTANT_Fieldref， CONSTANT_Methodref 和CONSTANT_InterfaceMethodref 结构处 name_and_type_index 项所引用的CONSTANT_NameAndType 结构得到。）

* 设 T 和（如果是方法的话） A* 是 R 所引用的字段或方法的返回值和参数类型序列（ T 和 A*是由得到 R 的 CONSTANT_Fieldref， CONSTANT_Methodref 和CONSTANT_InterfaceMethodref 结构处 name_and_type_index 项所引用的CONSTANT_NameAndType 结构得到。）

为了解析 MH，所有 MH 的字节码行为中对类、字段或方法的符号引用都必须进行解析（ §5.4.3.1、 §5.4.3.2、 §5.4.3.3、 §5.4.3.4）。 那就是说， C、 f、 m、 T 和 A*都需要解析。在解析这些与类、字段或方法相关的符号引用过程中所抛出的异常，都可以看作是解析方法句柄的异常而抛出。
（通常来说，成功解析一个方法句柄需要 Java 虚拟机事先成功解析字节码行为中的所有符号引用。特别的是，到 private 和 protected 成员的方法句柄可以被正常地创建使得相应地正常方法请求成为合法的请求）
如果所有这些符号引用被解析后就会获得一个对 java.lang.invoke.MethodType实例的引用，就像下表中所述的给定 MH 的类型而去解析方法描述符的行为一样 

| 类型   | 描述                   | 字节码行为   |
| ---- | -------------------- | ------- |
| 1    | REF_getField         | (C)T    |
| 2    | REF_getStatic        | ()T     |
| 3    | REF_putField         | (C,T)V  |
| 4    | REF_putStatic        | (T)V    |
| 5    | REF_invokeVirtual    | (C,A*)T |
| 6    | REF_invokeStatic     | (A*) T  |
| 7    | REF_invokeSpecial    | (C,A*)T |
| 8    | REF_newInvokeSpecial | (A*)C   |
| 9    | REF_invokeInterface  | (C,A*)T |

方法句柄解析的结果是一个指向 java.lang.invoke.MethodHandle 实例的引用 o，它表示着方法句柄 MH。如果方法 m 有 ACC_VARARGS 标志（ §4.6），那么 o 是一个可变元方法句柄；否则， o 是固定元方法句柄。
（可变元方法句柄在调用 invoke 方法时它的参数列表就会有装箱动作（ JLS §15.12.4.2），考虑 invokeExact 的调用行为就像 ACC_VARAGS 标志没有被设置一样。）
当方法 m 有 ACC_VARARGS 标志且要么 m 的参数类型序列为空，要么 m 参数类型的最后一个参数不是数组类型，那么方法句柄解析就会抛出 IncompatibleClassChangeError 异常（这表示创建可变元方法句柄失败）。
o 所引用的 java.lang.invoke.MethodHandle 实例的类型描述符是一个java.lang.invoke.MethodType 的实例，它是之前由方法类型解析时产生的。
（在方法句柄的类型描述符上调用 java.lang.invoke.MethodHandle.invokeExact方法会与字符码行一样，对栈造成相同的影响。当方法句柄带有有效的参数集合时，它会与相应的字节码行为有相同的影响及返回值）
（ Java 虚拟机的实现完全可以不需要内部的方法类型与方法句柄。那就是说，纵使拥有同样结构的对方法类型或方法句柄的两个不同符号引用都可能会获得不同的java.lang.invoke.MethodType 或 java.lang.invoide.MethodHandle 实例。）
（在 Java SE 平台 API 中，允许在没有字节码行为的时候通过java.lang.invoke.MethodHandles 类创建方法句柄。具体的行为是由创建java.lang.invoke.MethodHandles 的方法来决定。例如，当调用一个方法句柄时，就对传入它的参数，并通过传入的值去调用另外一个方法句柄，接着将调用的返回值传回来，并返回传回的值作为它的最终结果。） 




