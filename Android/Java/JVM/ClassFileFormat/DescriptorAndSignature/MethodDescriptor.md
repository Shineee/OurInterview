# 方法描述符


方法描述符（ Method Descriptor） 描述一个方法所需的参数和返回值信息：

```
MethodDescriptor:

( ParameterDescriptor* ) ReturnDescriptor
```

参数描述符（ ParameterDescriptor） 描述需要传给这个方法的参数信息：

```
ParameterDescriptor:

FieldType
```

返回描值述符(ReturnDescriptor)从当前方法返回的值，它是由语法产生的字符序列：

```
ReturnDescriptor:

FieldType

VoidDescriptor
```

其中 VoidDescriptor 表示当前方法无返回值，即返回类型是 void。符号如下（ 字符 V 即

```
void）：

VoidDescriptor:

V
```

如果一个方法描述符是有效的，那么它对应的方法的参数列表总长度小于等于 255，对于实
例方法和接口方法，需要额外考虑隐式参数 this。参数列表长度的计算规则如下：每个 long 和 

double 类参数长度为 2，其余的都为 1，方法参数列表的总长度等于所有参数的长度之和。
例如，方法：

```
Object mymethod(int i, double d, Thread t)
```

的描述符为：

```
(IDLjava/lang/Thread;)Ljava/lang/Object;
```

注意：这里使用了 Object 和 Thread 的二进制名称的内部形式。
无论 mymethod()是静态方法还是实例方法，它的方法描述符都是相同的。尽管实例方法除了传递自身定义的参数，还需要额外传递参数 this，但是这一点不是由法描述符来表达的。参数this 的传递，是由 Java 虚拟机实现在调用实例方法所使用的指令中实现的隐式传递。 






