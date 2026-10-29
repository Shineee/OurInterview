# CONSTANT_Class_info 结构 

CONSTANT_Class_info 结构用于表示类或接口，格式如下： 

```
CONSTANT_Class_info {

u1 tag;

u2 name_index;

}
```

CONSTANT_Class_info 结构的项的说明：

* tag

  CONSTANT_Class_info 结构的 tag 项的值为 CONSTANT_Class（ 7）。


* name_index

  name_index 项的值，必须是对常量池的一个有效索引。 常量池在该索引处的项必须是CONSTANT_Utf8_info（ §4.4.7） 结构， 代表一个有效的类或接口二进制名称的内部形式。

因为数组也是由对象表示，所以字节码指令 anewarray 和 multianewarray 可以通过常量池中的 CONSTANT_Class_info（ §4.4.1） 结构来引用类数组。对于这些数组， 类的名字就是数组类型的描述符，例如：
表现二维 int 数组类型

```
int
```

的名字是：

```
[[I
```

表示一维 Thread 数组类型

```
Thread[]
```

的名字是：

```
[Ljava/lang/Thread;
```

一个有效的数组类型描述符中描述的数组维度必须小于等于 255。 