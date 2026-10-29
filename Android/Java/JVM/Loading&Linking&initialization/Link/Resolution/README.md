# 解析

Java 虚拟机指令 anewarray、 checkcast、 getfield、 getstatic、 instanceof、nvokedynamic、 invokeinterface、 invokespecial、 invokestatic、 invokevirtual、ldc、 ldc_w、 multianewarray、 new、 putfield 和 putstatic 将符号引用指向运行时常量池。执行上述任何一条指令都需要对它的符号引用的进行解析。
解析（ Resolution）是根据运行时常量池的符号引用来动态决定具体的值的过程。
当碰到已经因某个 invokedynamic 指令而解析过的符号引用时，并不意味着对于其它invokedynamic 指令，相同的符号引用也被解析过。
但是对于上述的其它指令，当碰到某个因被其他指令引用而解析的符号引用时，就表示对于其它所有非 invokedynamic 的指令来说，相同的符号引用已经被解析过了。
（上面的内容暗示着，对于特定的某一条 invokedynamic 指令，它的解析过程会返回一个特定的值，这个值是一个与此 invokedynamic 指令相关联的调用点对象。）
虚拟机在解析过程中也可以尝试去重新解析之前已经成功解析过的符号引用。如果有这样的尝试动作，那么它总是会像之前一样解析成功，且总是返回此与引用初次解析时的结果相同的实体。
如果在某个符号引用解析过程中有错误发生，那么就应该在使用（直接或间接） 这个符号引用的代码处抛出 IncompatibleClassChangeError 或它的子类异常。
如果在虚拟机解析符号引用时，因为 LinkageError 或它的子类实例而导致失败，那么随后的每次试图对此引用的解析也总会抛出与第一次解析时相同的错误。
通过 invokedynamic 指令指定的调用点限定符的符号引用，在执行这条 invokedynamic指令被实际执行之前不能被提早解析。
如果解析某个 invokedynamic 指令的时候出错，引导方法在随后的尝试解析时就不会再被重新执行。
上述的某些指令在解析符号引用时，需要有额外的链接检查。譬如， getfield 指令为了成功解析所指的字段的符号引用，不仅得完成 5.4.3.2 节描述的字段解析步骤，还得检查这个字符是不是 static 的。如果这个字段是 static 的，就必须抛出链接时异常。 

为了让 invokedynamic 指令成功解析一个指向调用点限定符的符号引用，指定的引导方法就必须正常执行完成并返回一个合适的调用点对象。如果引导方法执行被中断或返回一个不合法的调用点对象， 那也必须抛出链接时异常。

链接时异常的产生是由于 Java 虚拟机指令被检查出未按照指令定义中所描述的语义来执行，或没有通过指令的解析过程。对于这些异常，尽管有可能是因为 Java 虚拟机指令执行的问题而导致的，但这类问题依然会被当作是解析失败的问题。

后续各章节将描述对于类或接口 D 的运行时常量池（ §5.1） 中的符号引用进行解析的过程。解析细节会因为符号引用类型的不同而有所区别。 

### [类与接口解析](ClassAndInterfaceAnalysis.md)

### [字段解析](FieldAnalysis.md)

### [普通方法解析](CommonMethodAnalysis.md)

### [接口方法解析](InterfaceMethodAnalysis.md)

### [方法类型与方法句型解析](MethodTypeAndMethodHandle.md)

### [调用点限定符解析](CallPointQualifierResolution.md)