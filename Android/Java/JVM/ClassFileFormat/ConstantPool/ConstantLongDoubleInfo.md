# CONSTANT_Long_info 和 CONSTANT_Double_info 结构 

CONSTANT_Long_info 和 CONSTANT_Double_info 结构表示 8 字节（ long 和 double）的数值常量：

```
CONSTANT_Long_info {

u1 tag;

u4 high_bytes;

u4 low_bytes;

}

CONSTANT_Double_info {

u1 tag;

u4 high_bytes;

u4 low_bytes;

} 
```

在Class 文件的常量池中，所有的 8 字节的常量都占两个表成员（项）的空间。如果一个CONSTANT_Long_info 或 CONSTANT_Double_info 结构的项在常量池中的索引为 n，则常量池中下一个有效的项的索引为 n+2， 此时常量池中索引为 n+1 的项有效但必须被认为不可用①。
CONSTANT_Long_info 和 CONSTANT_Double_info 结构各项的说明如下：

* tag

  CONSTANT_Long_info 结构的 tag 项的值是 CONSTANT_Long（ 5）。

  CONSTANT_Double_info 结构的 tag 项的值是 CONSTANT_Double（ 6）。

* high_bytes 和 low_bytes

  CONSTANT_Long_info 结构中的无符号的 high_bytes 和 low_bytes 项用于共同表示 long 型常量，构造形式为((long) high_bytes << 32) + low_bytes，high_bytes 和 low_bytes 都按照 Big-Endian 顺序存储。CONSTANT_Double_info 结构中的 high_bytes 和 low_bytes 共同按照 IEEE 754 双精度浮点格式（ §2.3.2）表示 double 常量的值。 high_bytes 和 low_bytes 都按照 Big-Endian 顺序存储。

  CONSTANT_Double_info 结构表示的值将按照下列方式来表示， high_bytes 和 

  low_bytes 首先被转换成一个 long 常量的 bits：

  * 如果 bits 值为 0x7ff0000000000000L，表示 double 值为正无穷。

  * 如果 bits 值为 0xfff0000000000000L，表示 double 值为负无穷。

  * 如果 bits 值在范围 0x7ff0000000000001L 到 0x7fffffffffffffffL 或者0xfff0000000000001L 到 0xffffffffffffffffL内，表示 double值为 NaN。

  * 在其它情况下，设 s、 e、 m，它们的值根据 bits 和如下公式计算：

    ```
    int s =((bits >> 63) == 0) ? 1 : -1;

    int e =(int)((bits >> 52) & 0x7ffL);

    long m =(e == 0) ?(bits & 0xfffffffffffffL) << 1 : (bits & 0xfffffffffffffL) | 0x10000000000000L;
    ```

则 double 的浮点值为数学表达式 s·m·2e – 1075 的计算结果。 






