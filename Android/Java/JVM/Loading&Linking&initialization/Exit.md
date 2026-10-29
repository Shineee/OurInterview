# JVM 退出

Java 虚拟机的退出条件一般是：某些线程调用 Runtime 类或 System 类的 exit 方法，或是 Runtime 类的 halt 方法，并且 Java 安全管理器也允许这些 exit 或 halt 操作。

除此之外， 在 JNI（ Java Native Interface）规范中还描述了当使用 JNI API 来加载

和卸载（ Load & Unload） Java 虚拟机时， Java 虚拟机的退出过程。 