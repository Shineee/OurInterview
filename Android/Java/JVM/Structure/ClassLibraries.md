# 类库

Java 虚拟机必须对不同平台下 Java 类库的实现提供充分的支持，因为其中有一些类库如果没有 Java 虚拟机的支持的话是根本无法实现的。

可能需要 Java 虚拟机特殊支持的类库包括有：

* 反射，譬如在 java.lang.reflect 包中的各个类和 java.lang.Class 类 

* 类和接口的加载和创建，最显而易见的例子就是 java.lang.ClassLoader 类

* 类和接口的链接和初始化，上一点的例子也适用于这点

* 安全， 譬如在 java.security 包中的各个类和 java.lang.SecurityManager 等其

  他类

* 多线程，譬如 java.lang.Thread 类

* 弱引用， 譬如在 java.lang.ref 包中的各个类

上面列举的几点意在简单说明一下而不是详细介绍这些类库，介绍这些类库和功能的详细信息已经超出了本书的范围，如果读者想了解这些类库，阅读 Java 平台类库的说明书可以获得想要的信息。 














































