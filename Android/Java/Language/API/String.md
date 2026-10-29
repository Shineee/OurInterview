# Java String 类

字符串广泛应用在Java编程中，在Java中字符串属于对象，Java提供了String类来创建和操作字符串。

## 创建字符串

创建字符串最简单的方式如下:

```java
String greeting = "Hello world!";
```

在代码中遇到字符串常量时，这里的值是"Hello world!"，编译器会使用该值创建一个String对象。

和其它对象一样，可以使用关键字和构造方法来创建String对象。

String类有11种构造方法，这些方法提供不同的参数来初始化字符串，比如提供一个字符数组参数:

```java
public class StringDemo{

   public static void main(String args[]){
      char[] helloArray = { 'h', 'e', 'l', 'l', 'o', '.'};
      String helloString = new String(helloArray);  
      System.out.println( helloString );
   }
}
```

**注意:**String类是不可改变的，所以你一旦创建了String对象，那它的值就无法改变了。 如果需要对字符串做很多修改，那么应该选择使用[StringBuffer & StringBuilder 类](http://www.runoob.com/java/java-stringbuffer.html)。

## 字符串长度

用于获取有关对象的信息的方法称为访问器方法。

String类的一个访问器方法是length()方法，它返回字符串对象包含的字符数。

## 连接字符串

String类提供了连接两个字符串的方法：

```java
string1.concat(string2);
```

返回string2连接string1的新字符串。也可以对字符串常量使用concat()方法，如：

```java
"My name is ".concat("Runoob");
```

更常用的是使用'+'操作符来连接字符串，如：

```java
"Hello," + " world" + "!"
```

## 创建格式化字符串

我们知道输出格式化数字可以使用printf()和format()方法。String类使用静态方法format()返回一个String对象而不是PrintStream对象。

String类的静态方法format()能用来创建可复用的格式化字符串，而不仅仅是用于一次打印输出。如下所示：

```java
System.out.printf("浮点型变量的的值为 " +
                  "%f, 整型变量的值为 " +
                  " %d, 字符串变量的值为 " +
                  "is %s", floatVar, intVar, stringVar);
```

你也可以这样写

```java
String fs;
fs = String.format("浮点型变量的的值为 " +
                   "%f, 整型变量的值为 " +
                   " %d, 字符串变量的值为 " +
                   " %s", floatVar, intVar, stringVar);
System.out.println(fs);
```

## String 方法

下面是String类支持的方法，更多详细，参看 [Java String API](http://www.runoob.com/manual/jdk1.6/java/lang/String.html) 文档:

| SN(序号) | 方法描述                                     |
| ------ | ---------------------------------------- |
| 1      | [char charAt(int index)](#user-content-charAt)返回指定索引处的 char 值。 |
| 2      | [int compareTo(Object o)](#user-content-compareTo)把这个字符串和另一个对象比较。 |
| 3      | [int compareTo(String anotherString)](#user-content-compareTo)按字典顺序比较两个字符串。 |
| 4      | [int compareToIgnoreCase(String str)](#user-content-compareToIgnoreCase)按字典顺序比较两个字符串，不考虑大小写。 |
| 5      | [String concat(String str)](#user-content-concat)将指定字符串连接到此字符串的结尾。 |
| 6      | [boolean contentEquals(StringBuffer sb)](#user-content-contentEquals)当且仅当字符串与指定的StringButter有相同顺序的字符时候返回真。 |
| 7      | [static String copyValueOf(char[] data)](#user-content-copyValueOf)返回指定数组中表示该字符序列的 String。 |
| 8      | [static String copyValueOf(char[] data, int offset, int count)](#user-content-copyValueOf)返回指定数组中表示该字符序列的 String。 |
| 9      | [boolean endsWith(String suffix)](#user-content-endsWith)测试此字符串是否以指定的后缀结束。 |
| 10     | [boolean equals(Object anObject)](#user-content-equals)将此字符串与指定的对象比较。 |
| 11     | [boolean equalsIgnoreCase(String anotherString)](#user-content-equalsIgnoreCase)将此 String 与另一个 String 比较，不考虑大小写。 |
| 12     | [byte getBytes()](#user-content-getBytes) 使用平台的默认字符集将此 String 编码为 byte 序列，并将结果存储到一个新的 byte 数组中。 |
| 13     | [byte getBytes(String charsetName)](#user-content-getBytes)使用指定的字符集将此 String 编码为 byte 序列，并将结果存储到一个新的 byte 数组中。 |
| 14     | [void getChars(int srcBegin, int srcEnd, char dst, int dstBegin)](#user-content-getChars)将字符从此字符串复制到目标字符数组。 |
| 15     | [int hashCode()](#user-content-hashCode)返回此字符串的哈希码。 |
| 16     | [int indexOf(int ch)](#user-content-indexOf)返回指定字符在此字符串中第一次出现处的索引。 |
| 17     | [int indexOf(int ch, int fromIndex)](#user-content-indexOf)返回在此字符串中第一次出现指定字符处的索引，从指定的索引开始搜索。 |
| 18     | [int indexOf(String str)](#user-content-indexOf) 返回指定子字符串在此字符串中第一次出现处的索引。 |
| 19     | [int indexOf(String str, int fromIndex)](#user-content-indexOf)返回指定子字符串在此字符串中第一次出现处的索引，从指定的索引开始。 |
| 20     | [String intern()](#user-content-intern) 返回字符串对象的规范化表示形式。 |
| 21     | [int lastIndexOf(int ch)](#user-content-lastIndexOf) 返回指定字符在此字符串中最后一次出现处的索引。 |
| 22     | [int lastIndexOf(int ch, int fromIndex)](#user-content-lastIndexOf)返回指定字符在此字符串中最后一次出现处的索引，从指定的索引处开始进行反向搜索。 |
| 23     | [int lastIndexOf(String str)](#user-content-lastIndexOf)返回指定子字符串在此字符串中最右边出现处的索引。 |
| 24     | [int lastIndexOf(String str, int fromIndex)](#user-content-lastIndexOf) 返回指定子字符串在此字符串中最后一次出现处的索引，从指定的索引开始反向搜索。 |
| 25     | [int length()](#user-content-length)返回此字符串的长度。 |
| 26     | [boolean matches(String regex)](#user-content-matches)告知此字符串是否匹配给定的正则表达式。 |
| 27     | [boolean regionMatches(boolean ignoreCase, int toffset, String other, int ooffset, int len)](#user-content-regionMatches)测试两个字符串区域是否相等。 |
| 28     | [boolean regionMatches(int toffset, String other, int ooffset, int len)](#user-content-regionMatches)测试两个字符串区域是否相等。 |
| 29     | [String replace(char oldChar, char newChar)](#user-content-replace)返回一个新的字符串，它是通过用 newChar 替换此字符串中出现的所有 oldChar 得到的。 |
| 30     | [String replaceAll(String regex, String replacement](#user-content-replaceAll)使用给定的 replacement 替换此字符串所有匹配给定的正则表达式的子字符串。 |
| 31     | [String replaceFirst(String regex, String replacement)](#user-content-replaceFirst) 使用给定的 replacement 替换此字符串匹配给定的正则表达式的第一个子字符串。 |
| 32     | [String split(String regex)](#user-content-split)根据给定正则表达式的匹配拆分此字符串。 |
| 33     | [String split(String regex, int limit)](#user-content-split)根据匹配给定的正则表达式来拆分此字符串。 |
| 34     | [boolean startsWith(String prefix)](#user-content-startsWith)测试此字符串是否以指定的前缀开始。 |
| 35     | [boolean startsWith(String prefix, int toffset)](#user-content-startsWith)测试此字符串从指定索引开始的子字符串是否以指定前缀开始。 |
| 36     | [CharSequence subSequence(int beginIndex, int endIndex)](#user-content-subSequence) 返回一个新的字符序列，它是此序列的一个子序列。 |
| 37     | [String substring(int beginIndex)](#user-content-substring)返回一个新的字符串，它是此字符串的一个子字符串。 |
| 38     | [String substring(int beginIndex, int endIndex)](#user-content-substring)返回一个新字符串，它是此字符串的一个子字符串。 |
| 39     | [char toCharArray()](#user-content-toCharArray)将此字符串转换为一个新的字符数组。 |
| 40     | [String toLowerCase()](#user-content-toLowerCase)使用默认语言环境的规则将此 String 中的所有字符都转换为小写。 |
| 41     | [String toLowerCase(Locale locale)](#user-content-toLowerCase) 使用给定 Locale 的规则将此 String 中的所有字符都转换为小写。 |
| 42     | [String toString()](#user-content-toString) 返回此对象本身（它已经是一个字符串！）。 |
| 43     | [String toUpperCase()](#user-content-toUpperCase)使用默认语言环境的规则将此 String 中的所有字符都转换为大写。 |
| 44     | [String toUpperCase(Locale locale)](#user-content-toUpperCase)使用给定 Locale 的规则将此 String 中的所有字符都转换为大写。 |
| 45     | [String trim()](#user-content-trim)返回字符串的副本，忽略前导空白和尾部空白。 |
| 46     | [static String valueOf(primitive data type x)](#user-content-valueOf)返回给定data type类型x参数的字符串表示形式。 |



# <a name="charAt">charAt()</a>

charAt() 方法用于返回指定索引处的字符。索引范围为从 0 到 length() - 1。

### 语法

```java
public char charAt(int index)
```

### 参数

- **index** -- 字符的索引。

### 返回值

返回指定索引处的字符。

# <a name="compareTo">compareTo()</a>

compareTo() 方法用于两种方式的比较：

- 字符串与对象进行比较。
- 按字典顺序比较两个字符串。

### 语法

```java
int compareTo(Object o)

int compareTo(String anotherString)
```

### 参数

- **o** -- 要比较的对象。
- **anotherString** -- 要比较的字符串。

### 返回值

- 如果参数字符串等于此字符串，则返回值 0；
- 如果此字符串小于字符串参数，则返回一个小于 0 的值；
- 如果此字符串大于字符串参数，则返回一个大于 0 的值。

# <a name="compareToIgnoreCase">compareToIgnoreCase()</a>

compareToIgnoreCase() 方法用于按字典顺序比较两个字符串，不考虑大小写。

### 语法

```java
int compareToIgnoreCase(String str)
```

### 参数

- **str** -- 要比较的字符串。

### 返回值

- 如果参数字符串等于此字符串，则返回值 0；
- 如果此字符串小于字符串参数，则返回一个小于 0 的值；
- 如果此字符串大于字符串参数，则返回一个大于 0 的值。

# <a name="concat">concat()</a>

concat() 方法用于将指定的字符串参数连接到字符串上。

### 语法

```java
public String concat(String s)
```

### 参数

- **s** -- 要连接的字符串。

### 返回值

返回连接后的新字符串。

# <a name="contentEquals">contentEquals()</a>

contentEquals() 方法用于将将此字符串与指定的 StringBuffer 比较。

### 语法

```java
public boolean contentEquals(StringBuffer sb)
```

### 参数

- **sb** -- 要与字符串比较的 StringBuffer。

### 返回值

如字符串与指定 StringBuffer 表示相同的字符序列，则返回 true；否则返回 false。

# <a name="copyValueOf">copyValueOf()</a>

copyValueOf() 方法有两种形式：

- **public static String copyValueOf(char[] data):** 返回指定数组中表示该字符序列的字符串。
- **public static String copyValueOf(char[] data, int offset, int count):** 返回指定数组中表示该字符序列的 字符串。

### 语法

```java
public static String copyValueOf(char[] data)

public static String copyValueOf(char[] data, int offset, int count)
```

### 参数

- **data** -- 字符数组。
- **offset** -- 子数组的初始偏移量。。
- **count** -- 子数组的长度。

### 返回值

如字符串与指定 StringBuffer 表示相同的字符序列，则返回 true；否则返回 false。

# <a name="endsWith">endsWith()</a>

endsWith() 方法用于测试字符串是否以指定的后缀结束。

### 语法

```java
public boolean endsWith(String suffix)
```

### 参数

- **suffix** -- 指定的后缀。

### 返回值

如果参数表示的字符序列是此对象表示的字符序列的后缀，则返回 true；否则返回 false。注意，如果参数是空字符串，或者等于此 String 对象（用 equals(Object) 方法确定），则结果为 true。

# <a name="equals">equals()</a>

equals() 方法用于将字符串与指定的对象比较。

### 语法

```java
public boolean equals(Object anObject)
```

### 参数

- **anObject** -- 与字符串进行比较的对象。

### 返回值

如果给定对象与字符串相等，则返回 true；否则返回 false。

# <a name="equalsIgnoreCase">equalsIgnoreCase()</a>

equalsIgnoreCase() 方法用于将字符串与指定的对象比较，不考虑大小写。

### 语法

```java
public boolean equalsIgnoreCase(String anotherString)
```

### 参数

- **anObject** -- 与字符串进行比较的对象。

### 返回值

如果给定对象与字符串相等，则返回 true；否则返回 false。

# <a name="getBytes">getBytes()</a>

getBytes() 方法有两种形式：

- **getBytes(String charsetName):**使用平台的默认字符集将字符串编码为 byte 序列，并将结果存储到一个新的 byte 数组中。
- **getBytes(): **使用平台的默认字符集将字符串编码为 byte 序列，并将结果存储到一个新的 byte 数组中。

### 语法

```java
public byte[] getBytes(String charsetName) throws UnsupportedEncodingException

public byte[] getBytes()
```

### 参数

- **charsetName** -- 支持的字符集名称。

### 返回值

返回 byte 数组。

# <a name="getChars">getChars()</a>

getChars() 方法将字符从字符串复制到目标字符数组。

### 语法

```java
public void getChars(int srcBegin, int srcEnd, char[] dst,  int dstBegin)
```

### 参数

- **srcBegin** -- 字符串中要复制的第一个字符的索引。
- **srcEnd** -- 字符串中要复制的最后一个字符之后的索引。
- **dst** -- 目标数组。
- **dstBegin** -- 目标数组中的起始偏移量。

### 返回值

没有返回值，但会抛出 IndexOutOfBoundsException 异常。

# <a name="hashCode">hashCode()</a>

hashCode() 方法用于返回字符串的哈希码。

字符串对象的哈希码根据以下公式计算：

```java
s[0]*31^(n-1) + s[1]*31^(n-2) + ... + s[n-1]
```

使用 int 算法，这里 s[i] 是字符串的第 i 个字符，n 是字符串的长度，^ 表示求幂。空字符串的哈希值为 0。

### 语法

```java
public int hashCode()
```

### 参数

- 无。

### 返回值

返回对象的哈希码值。

# <a name="indexOf">indexOf()</a>

indexOf() 方法有以下四种形式：

- **public int indexOf(int ch):** 返回指定字符在字符串中第一次出现处的索引，如果此字符串中没有这样的字符，则返回 -1。
- **public int indexOf(int ch, int fromIndex):** 返回指定字符在字符串中第一次出现处的索引，如果此字符串中没有这样的字符，则返回 -1。
- **int indexOf(String str):** 返回指定字符在字符串中第一次出现处的索引，如果此字符串中没有这样的字符，则返回 -1。
- **int indexOf(String str, int fromIndex):** 返回指定字符在字符串中第一次出现处的索引，如果此字符串中没有这样的字符，则返回 -1。

### 语法

```java
public int indexOf(int ch )

public int indexOf(int ch, int fromIndex)

int indexOf(String str)

int indexOf(String str, int fromIndex)
```

### 参数

- **ch** -- 字符。
- **fromIndex** -- 开始搜索的索引位置。
- **str** -- 要搜索的子字符串。

### 返回值

指定子字符串在字符串中第一次出现处的索引，从指定的索引开始。

# <a name="intern">intern()</a>

intern() 方法返回字符串对象的规范化表示形式。

它遵循以下规则：对于任意两个字符串 s 和 t，当且仅当 s.equals(t) 为 true 时，s.intern() == t.intern() 才为 true。

### 语法

```java
public String intern()
```

### 参数

- 无

### 返回值

一个字符串，内容与此字符串相同，但一定取自具有唯一字符串的池。

# <a name="lastIndexOf">lastIndexOf()</a>

lastIndexOf() 方法有以下四种形式：

- **public int lastIndexOf(int ch):** 返回指定字符在此字符串中最后一次出现处的索引，如果此字符串中没有这样的字符，则返回 -1。
- **public int lastIndexOf(int ch, int fromIndex):** 返返回指定字符在此字符串中最后一次出现处的索引，如果此字符串中没有这样的字符，则返回 -1。
- **public int lastIndexOf(String str):** 返回指定字符在此字符串中最后一次出现处的索引，如果此字符串中没有这样的字符，则返回 -1。
- **public int lastIndexOf(String str, int fromIndex):** 返回指定字符在此字符串中最后一次出现处的索引，如果此字符串中没有这样的字符，则返回 -1。

### 语法

```java
public int lastIndexOf(int ch)

public int lastIndexOf(int ch, int fromIndex)

public int lastIndexOf(String str)

public int lastIndexOf(String str, int fromIndex)
```

### 参数

- **ch** -- 字符。
- **fromIndex** -- 开始搜索的索引位置。
- **str** -- 要搜索的子字符串。

### 返回值

指定子字符串在字符串中第一次出现处的索引值。

# <a name="length">length()</a>

length() 方法用于返回字符串的长度。

长度等于字符串中 16 位 Unicode 代码单元的数量。

### 语法

```java
public int length()
```

### 参数

- 无

### 返回值

返回字符串长度。

# <a name="matches">matches()</a>

matches() 方法用于检测字符串是否匹配给定的正则表达式。

调用此方法的 str.matches(regex) 形式与以下表达式产生的结果完全相同：

```java
Pattern.matches(regex, str)
```

### 语法

```java
public boolean matches(String regex)
```

### 参数

- **regex** -- 匹配字符串的正则表达式。

### 返回值

在字符串匹配给定的正则表达式时，返回 true。

# <a name="regionMatches">regionMatches()</a>

regionMatches() 方法用于检测两个字符串在一个区域内是否相等。

### 语法

```java
public boolean regionMatches(int toffset,
                             String other,
                             int ooffset,
                             int len)

public boolean regionMatches(boolean ignoreCase,
                             int toffset,
                             String other,
                             int ooffset,
                             int len)
```

### 参数

- **ignoreCase** -- 如果为 true，则比较字符时忽略大小写。
- **toffset** -- 此字符串中子区域的起始偏移量。
- **other** -- 字符串参数。
- **toffset** -- 字符串参数中子区域的起始偏移量。
- **len** -- 要比较的字符数。

### 返回值

如果字符串的指定子区域匹配字符串参数的指定子区域，则返回 true；否则返回 false。是否完全匹配或考虑大小写取决于 ignoreCase 参数。

# <a name="replace">replace()</a>

replace() 方法通过用 newChar 字符替换字符串中出现的所有 oldChar 字符，并返回替换后的新字符串。

### 语法

```java
public String replace(char oldChar,
                      char newChar)
```

### 参数

- **oldChar** -- 原字符。
- **newChar** -- 新字符。

### 返回值

替换后生成的新字符串。

# <a name="replaceAll">replaceAll()</a>

replaceAll() 方法使用给定的参数 replacement 替换字符串所有匹配给定的正则表达式的子字符串。

### 语法

```java
public String replaceAll(String regex,
                         String replacement)
```

### 参数

- **regex** -- 匹配此字符串的正则表达式。
- **newChar** -- 用来替换每个匹配项的字符串。

### 返回值

成功则返回替换的字符串，失败则返回原始字符串。

# <a name="replaceFirst">replaceFirst()</a>

replaceFirst() 方法使用给定的参数 replacement 替换字符串第一个匹配给定的正则表达式的子字符串。

### 语法

```java
public String replaceFirst(String regex,
                           String replacement)
```

### 参数

- **regex** -- 匹配此字符串的正则表达式。
- **replacement** -- 用来替换第一个匹配项的字符串。

### 返回值

成功则返回替换的字符串，失败则返回原始字符串。

# <a name="split">split()</a>

split() 方法根据匹配给定的正则表达式来拆分字符串。

### 语法

```java
public String[] split(String regex,
                      int limit)
```

### 参数

- **regex** -- 正则表达式分隔符。
- **limit** -- 分割的份数。

### 返回值

成功则返回替换的字符串，失败则返回原始字符串。

# <a name="startsWith">startsWith()</a>

startsWith() 方法用于检测字符串是否以指定的前缀开始。

### 语法

```java
public boolean startsWith(String prefix, int toffset)

public boolean startsWith(String prefix)
```

### 参数

- **prefix** -- 前缀。
- **toffset** -- 字符串中开始查找的位置。

### 返回值

如果字符串以指定的前缀开始，则返回 true；否则返回 false。

# <a name="subSequence">subSequence()</a>

startsWith() 方法返回一个新的字符序列，它是此序列的一个子序列。

### 语法

```java
public CharSequence subSequence(int beginIndex, int endIndex)
```

### 参数

- **beginIndex** -- 起始索引（包括）。
- **endIndex** -- 结束索引（不包括）。

### 返回值

返回一个新的字符序列，它是此序列的一个子序列。

# <a name="substring">substring()</a>

substring() 方法返回字符串的子字符串。

### 语法

```java
public String substring(int beginIndex)

public String substring(int beginIndex, int endIndex)
```

### 参数

- **beginIndex** -- 起始索引（包括）。
- **endIndex** -- 结束索引（不包括）。

### 返回值

子字符串。

# <a name="toCharArray">toCharArray()</a>

toCharArray() 方法将字符串转换为字符数组。

### 语法

```java
public char[] toCharArray()
```

### 参数

- 无

### 返回值

字符数组。

# <a name="toLowerCase">toLowerCase()</a>

toLowerCase() 方法将字符串转换为小写。

### 语法

```java
public String toLowerCase()

public String toLowerCase(Locale locale)
```

### 参数

- 无

### 返回值

转换为小写的字符串。

# <a name="toString">toString()</a>

toString() 方法返回此对象本身（它已经是一个字符串）。

### 语法

```java
public String toString()
```

### 参数

- 无

### 返回值

字符串本身。

# <a name="toUpperCase">toUpperCase()</a>

toUpperCase() 方法将字符串小写字符转换为大写。

### 语法

```java
public String toUpperCase()

public String toUpperCase(Locale locale)
```

### 参数

- 无

### 返回值

字符转换为大写后的字符串。

# <a name="trim">trim()</a>

trim() 方法用于删除字符串的头尾空白符。

### 语法

```java
public String trim()
```

### 参数

- 无

### 返回值

删除头尾空白符的字符串。

# <a name="valueOf">valueOf()</a>

valueOf() 方法有以下几种不同形式：

- **valueOf(boolean b):** 返回 boolean 参数的字符串表示形式。.
- **valueOf(char c):** 返回 char 参数的字符串表示形式。
- **valueOf(char[] data):** 返回 char 数组参数的字符串表示形式。
- **valueOf(char[] data, int offset, int count):** 返回 char 数组参数的特定子数组的字符串表示形式。
- **valueOf(double d):** 返回 double 参数的字符串表示形式。
- **valueOf(float f):** 返回 float 参数的字符串表示形式。
- **valueOf(int i):** 返回 int 参数的字符串表示形式。
- **valueOf(long l):** 返回 long 参数的字符串表示形式。
- **valueOf(Object obj):** 返回 Object 参数的字符串表示形式。

### 语法

```java
static String valueOf(boolean b) 

static String valueOf(char c) 

static String valueOf(char[] data) 

static String valueOf(char[] data, int offset, int count) 

static String valueOf(double d) 

static String valueOf(float f) 

static String valueOf(int i)

static String valueOf(long l)

static String valueOf(Object obj) 
```

### 参数

- 指定类型参数。

### 返回值

删除头尾空白符的字符串。