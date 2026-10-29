# Kotlin 语言深挖（专家向）

> 不写入门语法。聚焦编译语义、运行时边界与工程陷阱——面试中"语言题"从这里拿分。

## 目录

- [第1页](1.md)：空安全（平台类型、安全调用链、Java 互操作注解）；data class（equals/copy/组件N 与陷阱）；object 与伴生对象 vs Java 静态；sealed 与 when 穷尽、API 演进
- [第2页](2.md)：inline / noinline / crossinline；reified 与类型擦除绕过；value class 装箱与性能边界；委托属性（lazy 线程安全、by map、自定义）；扩展函数静态解析与 this 优先级；contracts 一句话深度
- [第3页](3.md)：高阶函数与内联对性能的意义；Sequence 惰性 vs List 提前求值；协变/逆变在声明位的实际影响；与 Java 互操作十大坑（@JvmOverloads、静态、泛型、平台类型等）
