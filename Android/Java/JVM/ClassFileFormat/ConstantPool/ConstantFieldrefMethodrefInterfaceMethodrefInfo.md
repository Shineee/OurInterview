# CONSTANT_Fieldref_info，CONSTANT_Methodref_info 和

字段，方法和接口方法由类似的结构表示：
字段： 

```
CONSTANT_Fieldref_info {

u1 tag;

u2 class_index;

u2 name_and_type_index;

}
```

方法：

```
CONSTANT_Methodref_info {

u1 tag;

u2 class_index;

u2 name_and_type_index;

}
```

接口方法：

```
CONSTANT_InterfaceMethodref_info {

u1 tag;

u2 class_index;

u2 name_and_type_index;

}
```

这些结构各项的说明如下：

* tag

  CONSTANT_Fieldref_info 结构的 tag 项的值为 CONSTANT_Fieldref（ 9）。

  CONSTANT_Methodref_info 结构的 tag 项的值为 CONSTANT_Methodref（ 10）CONSTANT_InterfaceMethodref_info 结构的 tag 项的值为CONSTANT_InterfaceMethodref（ 11）。

* class_index

  class_index 项的值必须是对常量池的有效索引， 常量池在该索引处的项必须是CONSTANT_Class_info（ §4.4.1）结构，表示一个类或接口，当前字段或方法是这个类或接口的成员。

  CONSTANT_Methodref_info 结构的 class_index 项的类型必须是类（不能是接口）。CONSTANT_InterfaceMethodref_info 结构的 class_index 项的类型必须是接口（不能是类）。 CONSTANT_Fieldref_info 结构的 class_index 项的类型既可以是类也可以是接口。

* name_and_type_index

  name_and_type_index 项的值必须是对常量池的有效索引， 常量池在该索引处的项必 须是 CONSTANT_NameAndType_info（ §4.4.6）结构，它表示当前字段或方法的名字和描述符。

  在一个 CONSTANT_Fieldref_info 结构中，给定的描述符必须是字段描述符（ §4.3.2）。而 CONSTANT_Methodref_info 和CONSTANT_InterfaceMethodref_info 中给定的描述符必须是方法描述符（ §4.3.3）。

  如果一个 CONSTANT_Methodref_info 结构的方法名以“ <”（ '\u003c'）开头，则说明这个方法名是特殊的<init>，即这个方法是实例初始化方法（ §2.9），它的返回类型必须为空。 

