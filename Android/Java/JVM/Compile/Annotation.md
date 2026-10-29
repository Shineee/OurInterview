# 注解 

注解（ Annotation） 在 Class 文件中如何表示将在§4.7.16 和§4.7.17 中详细描述，这两节明确描述了在 Class 文件格式中，如何描述修饰类型、 字段和方法的注解。这里只描述关于包注解的一些额外规则。 

如果编译器遇到一个被注解的、声明为在运行时可见的包， 那编译器将会生成一个表示接口的Class 文件， 内部形式为（ §4.2.1）“ package-name.package-info”。 这个接口有默认的访问权限（“ package-private”），并且没有父接口。 它的 ClassFile 结构（ §4.1）中的ACC_INTERFACE 和 ACC_ABSTRACT 的标志（ 表 4.1）会被自动设置。如果 Class 文件的版本号小于 50.0，则不设置 ACC_SYNTHETIC 标志； 如果 Class 文件的版本号为 50.0 或更高，则必须设置 ACC_SYNTHETIC 标志。接口中的全部成员都在《 Java 语言规范（ Java SE 7 版）》（ JLS§9.2）中被明确定义。
package 层次的注解中保存了 Class 文件结构（ §4.1） 中的RuntimeVisibleAnnotations（ §4.7.16） 和 RuntimeInvisibleAnnotations（ §4.7.17） 属性。 




