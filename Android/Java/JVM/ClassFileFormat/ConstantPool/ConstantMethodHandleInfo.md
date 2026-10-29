# CONSTANT_MethodHandle_info 结构 

CONSTANT_MethodHandle_info 结构用于表示方法句柄，结构如下：

```
CONSTANT_MethodHandle_info { 

u1 tag;

u1 reference_kind;

u2 reference_index;

}
```

CONSTANT_MethodHandle_info 结构各项的说明如下：

* tag

  CONSTANT_MethodHandle_info 结构的 tag 项的值为 CONSTANT_MethodHandle（ 15）。

* reference_kind

  reference_kind 项的值必须在 1 至 9 之间（包括 1 和 9），它决定了方法句柄的类型。

  方法句柄类型的值表示方法句柄的字节码行为（ Bytecode Behavior §5.4.3.5）。

* reference_index

  reference_index 项的值必须是对常量池的有效索引：

  * 如果 reference_kind 项的值为 1（ REF_getField）、 2（ REF_getStatic）、 3（ REF_putField）或 4（ REF_putStatic），那么常量池在 reference_index索引处的项必须是 CONSTANT_Fieldref_info（ §4.4.2）结构，表示由一个字段创建的方法句柄。
  * 如果 reference_kind 项的值是 5（ REF_invokeVirtual）、 6（ REF_invokeStatic）、 7（REF_invokeSpecial）或 8（ REF_newInvokeSpecial），那么常量池在 reference_index 索引处的项必须是 CONSTANT_Methodref_info（ §4.4.2）结构，表示由类的方法或构造函数创建的方法句柄。
  * 如果 reference_kind 项的值是 9（ REF_invokeInterface），那么常量池在reference_index索引处的项必须是 CONSTANT_InterfaceMethodref_info（ §4.4.2）结构，表示由接口方法创建的方法句柄。
  * 如果 reference_kind 项的值是 5（ REF_invokeVirtual）、 6（ REF_invokeStatic）、 7（ REF_invokeSpecial）或 9（ REF_invokeInterface），那么方法句柄对应的方法不能为实例初始化 （ <init>）方法或类初始化方法（ <clinit>）。
  * 如果 reference_kind 项的值是 8（ REF_newInvokeSpecial），那么方法句柄 对应的方法必须为实例初始化（ <init>）方法。 