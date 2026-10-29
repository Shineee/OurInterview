# 调用点限定符解析

解析一个未被解析过的调用点限定符（ Call Site Specifier）需要下列三个步骤：

* 调用点限定符提供对方法句柄的符号引用，它作为引导方法（ Bootstrap Method）向动态调用点提供服务。解析这个方法句柄（ §5.4.3.5）是为了获取一个对java.lang.invoke.MethodHandle 实例的引用。

* 调用点限定符提供了一个方法描述符，记作 TD。它是一个java.lang.invoke.MethodType 实例的引用。可以通过解析与 TD 有相同的参数及返回值的方法类型（ §5.4.3.5）的符号引用而获得。

* 调用点限定符提供零至多个静态参数（ Static Arguments），用于传递与特定应用相关的元数据给引导方法。静态参数只要是对类、方法句柄或方法类型的符号引用，都需要被解析，例如调用 ldc 指令，可以分别获取到对 Class 对象、java.lang.invoke.MethodHandle 对象和 java.lang.invoke.MethodType 对象等。如果静态参数都是字面常量，它就是为了获取对 String 对象的引用。

  调用点限定符的解析结果是一个数组，它包含：

  * 到 java.lang.invoke.MethodHandle 实例的引用，
  * 到 java.lang..invoke.MethodType 实例的引用，
  * 到 Class， java.lang.invoke.MethodHandle，java.lang.invoke.MethodType 和 String 实例的引用。

在解析调用点限定符的方法句柄符号引用，或解析调用点限定符中方法类型的描述符的符号引用时，或是解析任何的静态参数的符号引用时，任何与方法类型或方法句柄解析（ §5.4.3.5）有关的异常都可以被抛出。 