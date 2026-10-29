# Java Character类

Character 类用于对单个字符进行操作。

Character 类在对象中包装一个基本类型 **char** 的值

### 实例

```
char ch = 'a';

// Unicode 字符表示形式
char uniChar = '\u039A'; 

// 字符数组
char[] charArray ={ 'a', 'b', 'c', 'd', 'e' }; 
```

然而，在实际开发过程中，我们经常会遇到需要使用对象，而不是内置数据类型的情况。为了解决这个问题，Java语言为内置数据类型char提供了包装类Character类。

Character类提供了一系列方法来操纵字符。你可以使用Character的构造方法创建一个Character类对象，例如：

```
Character ch = new Character('a');
```

在某些情况下，Java编译器会自动创建一个Character对象。

例如，将一个char类型的参数传递给需要一个Character类型参数的方法时，那么编译器会自动地将char类型参数转换为Character对象。 这种特征称为装箱，反过来称为拆箱。

### 实例

```
// 原始字符 'a' 装箱到 Character 对象 ch 中
Character ch = 'a';

// 原始字符 'x' 用 test 方法装箱
// 返回拆箱的值到 'c'
char c = test('x');
```

------

## 转义序列

前面有反斜杠（\）的字符代表转义字符，它对编译器来说是有特殊含义的。

下面列表展示了Java的转义序列：

| 转义序列 | 描述            |
| ---- | ------------- |
| \t   | 在文中该处插入一个tab键 |
| \b   | 在文中该处插入一个后退键  |
| \n   | 在文中该处换行       |
| \r   | 在文中该处插入回车     |
| \f   | 在文中该处插入换页符    |
| \'   | 在文中该处插入单引号    |
| \"   | 在文中该处插入双引号    |
| \\   | 在文中该处插入反斜杠    |

## Character 方法

下面是Character类的方法：

| 序号   | 方法与描述                                    |
| ---- | ---------------------------------------- |
| 1    | [isLetter()](#user-content-isLetter)是否是一个字母 |
| 2    | [isDigit()](#user-content-isDigit)是否是一个数字字符 |
| 3    | [isWhitespace()](#user-content-isWhitespace)是否一个空格 |
| 4    | [isUpperCase()](#user-content-isUpperCase)是否是大写字母 |
| 5    | [isLowerCase()](#user-content-isLowerCase)是否是小写字母 |
| 6    | [toUpperCase()](#user-content-toUpperCase)指定字母的大写形式 |
| 7    | [toLowerCase()](#user-content-toLowerCase)指定字母的小写形式 |
| 8    | [toString()](#user-content-toString)返回字符的字符串形式，字符串的长度仅为1 |



# <a name="compareTo">isLetter()</a>

isLetter() 方法用于判断指定字符是否为字母。

### 语法

```java
boolean isLetter(char ch)
```

### 参数

- **ch** -- 要测试的字符。

### 返回值

如果字符为字母，则返回 true；否则返回 false。

# <a name="isDigit">isDigit()</a>

isDigit() 方法用于判断指定字符是否为数字。

### 语法

```java
boolean isDigit(char ch)
```

### 参数

- **ch** -- 要测试的字符。

### 返回值

如果字符为数字，则返回 true；否则返回 false。

# <a name="isWhitespace">isWhitespace()</a>

isWhitespace() 方法用于判断指定字符是否为空白字符，空白符包含：空格、tab键、换行符。

### 语法

```java
boolean isWhitespace(char ch)
```

### 参数

- **ch** -- 要测试的字符。

### 返回值

如果字符为空白字符，则返回 true；否则返回 false。

# <a name="isLowerCase">isLowerCase()</a>

isLowerCase() 方法用于判断指定字符是否为小写字母。

### 语法

```java
boolean isLowerCase(char ch)
```

### 参数

- **ch** -- 要测试的字符。

### 返回值

如果字符为小写，则返回 true；否则返回 false。

# <a name="isUpperCase">isUpperCase()</a>

isUpperCase() 方法用于判断指定字符是否为大写字母。

### 语法

```java
boolean isUpperCase(char ch)
```

### 参数

- **ch** -- 要测试的字符。

### 返回值

如果字符为大写，则返回 true；否则返回 false。

# <a name="toLowerCase">toLowerCase()</a>

toLowerCase() 方法用于将大写字符转换为小写。

### 语法

```java
char toLowerCase(char ch)
```

### 参数

- **ch** -- 要转换的字符。

### 返回值

返回转换后字符的小写形式，如果有的话；否则返回字符本身。

# <a name="toUpperCase">toUpperCase()</a>

toUpperCase() 方法用于将小写字符转换为大写。

### 语法

```java
char toUpperCase(char ch)
```

### 参数

- **ch** -- 要转换的字符。

### 返回值

返回转换后字符的大写形式，如果有的话；否则返回字符本身。

# <a name="toString">toString()</a>

toString() 方法用于返回一个表示指定 char 值的 String 对象。结果是长度为 1 的字符串，仅由指定的 char 组成。

### 语法

```java
String toString(char ch)
```

### 参数

- **ch** -- 要转换的字符。

### 返回值

返回指定 char 值的字符串表示形式。