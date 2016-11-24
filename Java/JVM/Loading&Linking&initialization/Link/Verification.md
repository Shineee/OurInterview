# 验证

验证（ Verification， §4.10）阶段用于确保类或接口的二进制表示结构上是正确的

（ §4.9）。验证过程可能会导致某些额外的类和接口被加载进来（ §5.3），但不应该会导致它们也需要验证或准备。
如果类或接口的二进制表示不能满足 4.9 节（ Java 虚拟机代码限制）中描述的静态或结构上的约束，那就必须在导致验证发生的程序调用处被抛出 VerityError 异常。
如果 Java 虚拟机尝试验证类或接口，却因为 LinkageError 或其子类的实例而导致验证失败，那么随后对于此类或接口的验证尝试总是会因为第一次尝试失败的同样原因而失败。 