# JVM Class 文件格式

本章将描述 Java 虚拟机中定义的 Class 文件格式。每一个 Class 文件都对应着唯一一个类或接口的定义信息，但是相对地， 类或接口并不一定都得定义在文件里（ 譬如类或接口也可以通过类加载器直接生成）。 本章中，我们只是通俗地将任意一个有效的类或接口所应当满足的格式称为“ Class 文件格式”， 即使它不一定以磁盘文件的形式存在。

每个 Class 文件都是由 8 字节为单位的字节流组成，所有的 16 位、 32 位和 64 位长度的数据将被构造成 2 个、 4 个和 8 个 8 字节单位来表示。多字节数据项总是按照 Big-Endian①的顺序进行存储。在 Java SDK 中，访问这种格式的数据可以使用 java.io.DataInput、java.io.DataOutput 等接口和 java.io.DataInputStream 和 java.io.DataOutputStream 等类来实现。
本章还定义了一组私有数据类型来表示 Class 文件的内容，它们包括 u1， u2 和 u4，分别代表了 1、 2 和 4 个字节的无符号数。在 Java SDK 中这些类型的数据可以通过实现接口java.io.DataInput 中的 readUnsignedByte、 readUnsignedShort 和 readInt 方法进行读取。
本章将采用类似 C 语言结构体的伪结构来描述 Class 文件格式。为了避免与类的字段、 类的实例等概念产生混淆， 在此把用于描述类结构格式的内容定义为项（ Item）。在 Class 文件中，各项按照严格顺序连续存放的， 它们之间没有任何填充或对齐作为各项间的分隔符号。
表（ Table） 是由任意数量的可变长度的项组成，用于表示 Class 文件内容的一系列复合结构。尽管我们采用类似 C 语言的数组语法来表示表中的项，但是读者应当清楚意识到， 表是由可变长数据组成的复合结构（ 表中每项的长度不固定），因此无法直接将字节偏移量来作为索引对表进行访问。 而我们描述一个数据结构为数组（ Array） 时，就意味着它含有零至多个长度固定的项组成，这个时候则可以采用数组索引的方式来访问它②。 

### [ClassFile 结构](ClassFileStructure.md)

### [各种内部表示名称](InternalRepresentationName)

### [描述符和签名](DescriptorAndSignature)

### [常量池](ConstantPool)

### [字段](Field.md)

### [方法](Method.md)

### [属性](Attributes)

### [格式检查](FormatCheck.md)

### [Java虚拟机代码约束](VirtualMachineCodeConstraints)

### [Class 文件校验](ClassHashtab)

### [Java 虚拟机限制](Limit.md)