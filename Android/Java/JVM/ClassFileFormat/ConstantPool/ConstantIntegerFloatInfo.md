# CONSTANT_Integer_info 和 CONSTANT_Float_info 结构 


CONSTANT_Intrger_info 和 CONSTANT_Float_info 结构表示 4 字节（ int 和 float）
的数值常量： 

```
CONSTANT_Integer_info {

u1 tag;

u4 bytes;

}

CONSTANT_Float_info {

u1 tag;

u4 bytes;

}
```

这些结构各项的说明如下：

* tag

  CONSTANT_Integer_info 结构的 tag 项的值是 CONSTANT_Integer（ 3）。CONSTANT_Float_info 结构的 tag 项的值是 CONSTANT_Float（ 4）。

* bytes

  CONSTANT_Integer_info 结构的 bytes 项表示 int 常量的值，按照 Big-Endian的顺序存储。

  CONSTANT_Float_info 结构的 bytes 项按照 IEEE 754 单精度浮点格式 （ §2.3.2）.

  表示 float 常量的值，按照 Big-Endian 的顺序存储。

  CONSTANT_Float_info 结构表示的值将按照下列方式来表示， bytes 项的值首先被转换成一个 int 常量的 bits：

  * 如果 bits 值为 0x7f800000，表示 float 值为正无穷。

  * 如果 bits 值为 0xff800000，表示 float 值为负无穷。

  * 如果 bits 值在范围 0x7f800001 到 0x7fffffff 或者 0xff800001 到0xffffffff 内，表示 float 值为 NaN。

  * 在其它情况下，设 s、 e、 m，它们值根据 bits 和如下公式计算：

    ```
    int s =((bits >> 31) == 0) ? 1 : -1;

    int e =((bits >> 23) & 0xff);

    int m =(e == 0） ?bits & 0x7fffff) << 1 :(bits & 0x7fffff) | 0x800000;
    ```

则 float 的浮点值为数值表达式 s·m·2e–150的计算结果。 


