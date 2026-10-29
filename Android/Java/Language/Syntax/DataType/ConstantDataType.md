# Java 引用数据类型
常量在程序运行时，不会被修改的量。在 Java 中使用 final 关键字来修饰常量。

```java
final double PI = 3.1415927;
```

虽然常量名也可以用小写，但为了便于识别，通常使用大写字母表示常量。

字面量可以赋给任何内置类型的变量。

```java
byte a = 68;
char a = 'A'
```

byte、int、long、和short都可以用十进制、16进制以及8进制的方式来表示。

当使用常量的时候，前缀0表示8进制，而前缀0x代表16进制。

```java
int decimal = 100;
int octal = 0144;
int hexa =  0x64;
```

和其他语言一样，Java的字符串常量也是包含在两个引号之间的字符序列。

```java
"Hello World"
"two\nlines"
"\"This is in quotes\""
```

字符串常量和字符常量都可以包含任何Unicode字符。

```java
char a = '\u0001';
String a = "\u0001";
```

Java语言支持一些特殊的转义字符序列。

| 符号     | 字符含义                 |
| ------ | -------------------- |
| \n     | 换行 (0x0a)            |
| \r     | 回车 (0x0d)            |
| \f     | 换页符(0x0c)            |
| \b     | 退格 (0x08)            |
| \s     | 空格 (0x20)            |
| \t     | 制表符                  |
| \"     | 双引号                  |
| \'     | 单引号                  |
| \\     | 反斜杠                  |
| \ddd   | 八进制字符 (ddd)          |
| \uxxxx | 16进制Unicode字符 (xxxx) |

