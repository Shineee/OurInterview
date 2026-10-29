# Java Number类

一般地，当需要使用数字的时候，我们通常使用内置数据类型，如：byte、int、long、double等。

然而，在实际开发过程中，我们经常会遇到需要使用对象，而不是内置数据类型的情形。为了解决这个问题，Java语言为每一个内置数据类型提供了对应的包装类。

所有的包装类（Integer、Long、Byte、Double、Float、Short）都是抽象类Number的子类。

这种由编译器特别支持的包装称为装箱，所以当内置数据类型被当作对象使用的时候，编译器会把内置类型装箱为包装类。相似的，编译器也可以把一个对象拆箱为内置类型。Number类属于java.lang包。

### Number 方法

下面的表中列出的是 Number 子类实现的方法：

| 序号   | 方法与描述                                    |
| ---- | ---------------------------------------- |
| 1    | [xxxValue()](#user-content-xxxValue)将number对象转换为xxx数据类型的值并返回。 |
| 2    | [compareTo()](#user-content-compareTo)将number对象与参数比较。 |
| 3    | [equals()](#user-content-equals)判断number对象是否与参数相等。 |
| 4    | [valueOf()](#user-content-valueOf)返回一个 Number 对象指定的内置数据类型 |
| 5    | [toString()](#user-content-toString)以字符串形式返回值。 |
| 6    | [parseInt()](#user-content-parseInt)将字符串解析为int类型。 |
| 7    | [abs()](#user-content-abs)返回参数的绝对值。      |
| 8    | [ceil()](#user-content-ceil)对整形变量向左取整，返回类型为double型。 |
| 9    | [floor()](#user-content-floor)对整型变量向右取整。返回类型为double类型。 |
| 10   | [rint()](#user-content-rint)返回与参数最接近的整数。返回类型为double。 |
| 11   | [round()](#user-content-round)返回一个最接近的int、long型值。 |
| 12   | [min()](#user-content-min)返回两个参数中的最小值。   |
| 13   | [max()](#user-content-max)返回两个参数中的最大值。   |
| 14   | [exp()](#user-content-exp)返回自然数底数e的参数次方。 |
| 15   | [log()](#user-content-log)返回参数的自然数底数的对数值。 |
| 16   | [pow()](#user-content-pow)返回第一个参数的第二个参数次方。 |
| 17   | [sqrt()](#user-content-sqrt)求参数的算术平方根。   |
| 18   | [sin()](#user-content-sin)求指定double类型参数的正弦值。 |
| 19   | [cos()](#user-content-cos)求指定double类型参数的余弦值。 |
| 20   | [tan()](#user-content-tan)求指定double类型参数的正切值。 |
| 21   | [asin()](#user-content-asin)求指定double类型参数的反正弦值。 |
| 22   | [acos()](#user-content-acos)求指定double类型参数的反余弦值。 |
| 23   | [atan()](#user-content-atan)求指定double类型参数的反正切值。 |
| 24   | [atan2()](#user-content-atan2)将笛卡尔坐标转换为极坐标，并返回极坐标的角度值。 |
| 25   | [toDegrees()](#user-content-toDegrees)将参数转化为角度。 |
| 26   | [toRadians()](#user-content-toRadians)将角度转换为弧度。 |
| 27   | [random()](#user-content-random)返回一个随机数。 |



# <a name="xxxValue">xxxValue()</a>



xxxValue() 方法用于将 Number 对象转换为 **xxx ** 数据类型的值并返回。

相关的方法有：

| 类型              | 方法及描述                                  |
| --------------- | -------------------------------------- |
| byte            | **byteValue() :**以 byte 形式返回指定的数值。     |
| abstract double | **doubleValue() :**以 double 形式返回指定的数值。 |
| abstract float  | **floatValue() :**以 float 形式返回指定的数值。   |
| abstract int    | **intValue() :**以 int 形式返回指定的数值。       |
| abstract long   | **longValue() :**以 long 形式返回指定的数值。     |
| short           | **shortValue() :**以 short 形式返回指定的数值。   |

### 参数

以上各函数不接受任何的参数。

### 返回值

转换为 **xxx** 类型后该对象表示的数值。

# <a name="compareTo">compareTo()</a>

ompareTo() 方法用于将 Number 对象与方法的参数进行比较。可用于比较 Byte, Long, Integer等。

该方法用于两个相同数据类型的比较，两个不同类型的数据不能用此方法来比较。

### 语法

```java
public int compareTo( NumberSubClass referenceName )
```

### 参数

**referenceName** -- 可以是一个 Byte, Double, Integer, Float, Long 或 Short 类型的参数。

### 返回值

- 如果指定的数与参数相等返回0。
- 如果指定的数小于参数返回 -1。
- 如果指定的数大于参数返回 1。

# <a name="equals">equals()</a>

equals() 方法用于判断 Number 对象与方法的参数进是否相等。

### 语法

```java
public boolean equals(Object o)
```

### 参数

**o** -- 任何对象。

### 返回值

如 Number 对象不为 Null，且与方法的参数类型与数值都相等返回 True，否则返回 False。

Double 和 Float 对象还有一些额外的条件，可以参阅 API 手册：[JDK 1.6](http://www.runoob.com/manual/jdk1.6/)。

# <a name="valueOf">valueOf()</a>

valueOf() 方法用于返回给定参数的原生 Number 对象值，参数可以是原生数据类型, String等。

该方法是静态方法。该方法可以接收两个参数一个是字符串，一个是基数。

### 语法

该方法有以下几种语法格式：

```java
static Integer valueOf(int i)
static Integer valueOf(String s)
static Integer valueOf(String s, int radix)
```

### 参数

- **i** -- Integer 对象的整数。
- **s** -- Integer 对象的字符串。
- **radix** --在解析字符串 s 时使用的基数，用于指定使用的进制数。

### 返回值

- **Integer valueOf(int i)：**返回一个表示指定的 int 值的 Integer 实例。
- **Integer valueOf(String s):**返回保存指定的 String 的值的 Integer 对象。
- **Integer valueOf(String s, int radix):** 返回一个 Integer 对象，该对象中保存了用第二个参数提供的基数进行解析时从指定的 String 中提取的值。

# <a name="toString">toString()</a>

valueOf() 方法用于返回以一个字符串表示的 Number 对象值。

如果方法使用了原生的数据类型作为参数，返回原生数据类型的 String 对象值。

如果方法有两个参数， 返回用第二个参数指定基数表示的第一个参数的字符串表示形式。

### 语法

以 String 类为例，该方法有以下几种语法格式：

```java
String toString()
static String toString(int i)
```

### 参数

- **i** -- 要转换的整数。

### 返回值

- **toString():** 返回表示 Integer 值的 String 对象。
- **toString(int i):** 返回表示指定 int 的 String 对象。

# <a name="parseInt">parseInt()</a>

parseInt() 方法用于将字符串参数作为有符号的十进制整数进行解析。

如果方法有两个参数， 使用第二个参数指定的基数，将字符串参数解析为有符号的整数。

### 语法

所有 Number 派生类 parseInt 方法格式类似如下：

```java
static int parseInt(String s)

static int parseInt(String s, int radix)
```

### 参数

- **s** -- 十进制表示的字符串。
- **radix** -- 指定的基数。

### 返回值

- **parseInt(String s):** 返回用十进制参数表示的整数值。
- **parseInt(int i):** 使用指定基数的字符串参数表示的整数 (基数可以是 10, 2, 8, 或 16 等进制数) 。

# <a name="abs">abs()</a>

abs() 返回参数的绝对值。参数可以是 int, float, long, double, short, byte类型。

### 语法

各个类型的方法格式类似如下：

```java
double abs(double d)
float abs(float f)
int abs(int i)
long abs(long lng)
```

### 参数

- 任何原生数据类型。

### 返回值

返回参数的绝对值。

# <a name="ceil">ceil()</a>

ceil() 方法可对一个数进行上舍入，返回值大于或等于给定的参数。

### 语法

该方法有以下几种语法格式：

```java
double ceil(double d)

double ceil(float f)
```

### 参数

- double 或 float 的原生数据类型。

### 返回值

返回 double 类型，返回值大于或等于给定的参数。

# <a name="floor">floor()</a>

floor() 方法可对一个数进行下舍入，返回给定参数最大的整数，该整数小于或等给定的参数。

### 语法

该方法有以下几种语法格式：

```java
double floor(double d)

double floor(float f)
```

### 参数

- double 或 float 的原生数据类型。

### 返回值

返回 double 类型数组，小于或等于给定的参数。

# <a name="rint">rint()</a>

rint() 方法返回最接近参数的整数值。

### 语法

该方法有以下几种语法格式：

```java
double rint(double d)
```

### 参数

- double 原始数据类型。

### 返回值

返回 double 类型数组，是最接近参数的整数值。

# <a name="round">round()</a>

rint() 方法返回一个最接近的int、long型值。

### 语法

该方法有以下几种语法格式：

```java
long round(double d)

int round(float f)
```

### 参数

- **d **-- double 或 float 的原生数据类型
- **f **-- float 原生数据类型

### 返回值

返回一个最接近的int、long型值，方法会指定返回的数据类型。

# <a name="min">min()</a>

min() 方法用于返回两个参数中的最小值。

### 语法

该方法有以下几种语法格式：

```java
double min(double arg1, double arg2)
float min(float arg1, float arg2)
int min(int arg1, int arg2)
long min(long arg1, long arg2)
```

### 参数

该方法接受两个原生数据类型作为参数。

### 返回值

返回两个参数中的最小值。

# <a name="max">max()</a>

max() 方法用于返回两个参数中的最大值。

### 语法

该方法有以下几种语法格式：

```java
double max(double arg1, double arg2)
float max(float arg1, float arg2)
int max(int arg1, int arg2)
long max(long arg1, long arg2)
```

### 参数

该方法接受两个原生数据类型作为参数。

### 返回值

返回两个参数中的最大值。

# <a name="exp">exp()</a>

exp() 方法用于返回自然数底数e的参数次方。

### 语法

该方法有以下几种语法格式：

```java
double exp(double d)
```

### 参数

- **d** -- 任何原生数据类型。

### 返回值

返回自然数底数e的参数次方。

# <a name="log">log()</a>

log() 方法用于返回参数的自然数底数的对数值。

### 语法

```java
double log(double d)
```

### 参数

- **d** -- 任何原生数据类型。

### 返回值

返回参数的自然数底数的对数值。

# <a name="pow">pow()</a>

pow() 方法用于返回第一个参数的第二个参数次方。

### 语法

```java
double pow(double base, double exponent)
```

### 参数

- **base** -- 任何原生数据类型。
- **exponent** -- 任何原生数据类型。

### 返回值

返回第一个参数的第二个参数次方。

# <a name="sqrt">sqrt()</a>

sqrt() 方法用于返回参数的算术平方根。

### 语法

```java
double sqrt(double d)
```

### 参数

- **d **-- 任何原生数据类型。

### 返回值

返回参数的算术平方根。

# <a name="sin">sin()</a>

sin() 方法用于返回指定double类型参数的正弦值。

### 语法

```java
double sin(double d)
```

### 参数

- **d **-- 任何原生数据类型。

### 返回值

返回指定double类型参数的正弦值。

# <a name="cos">cos()</a>

cos() 方法用于返回指定double类型参数的余弦值。

### 语法

```java
double cos(double d)
```

### 参数

- **d **-- 任何原生数据类型。

### 返回值

返回指定double类型参数的余弦值。

# <a name="tan">tan()</a>

tan() 方法用于返回指定double类型参数的正切值。

### 语法

```java
double tan(double d)
```

### 参数

- **d **-- 任何原生数据类型。

### 返回值

返回指定double类型参数的正切值。

# <a name="asin">asin()</a>

asin() 方法用于返回指定double类型参数的反正弦值。

### 语法

```java
double asin(double d)
```

### 参数

- **d **-- 任何原生数据类型。

### 返回值

返回指定double类型参数的反正弦值。

# <a name="acos">acos()</a>

acos() 方法用于返回指定double类型参数的反余弦值。

### 语法

```java
double acos(double d)
```

### 参数

- **d **-- 任何原生数据类型。

#### 返回值

返回指定double类型参数的反余弦值。

# <a name="atan">atan()</a>

atan() 方法用于返回指定double类型参数的反正切值。

### 语法

```java
double atan(double d)
```

### 参数

- **d **-- 任何原生数据类型。

### 返回值

返回指定double类型参数的反正切值。

# <a name="atan2">atan2()</a>

atan2() 方法用于将矩形坐标 (x, y) 转换成极坐标 (r, theta)，返回所得角 theta。该方法通过计算 y/x 的反正切值来计算相角 theta，范围为从 -pi 到 pi。

### 语法

```java
double atan2(double y, double x)
```

### 参数

- **y** -- 纵坐标。
- **x** -- 横坐标。

### 返回值

与笛卡儿坐标中点 (x, y) 对应的极坐标中点 (r, theta) 的 theta 组件。

# <a name="toDegrees">toDegrees()</a>

toDegrees() 方法用于将参数转化为角度。

### 语法

```java
double toDegrees(double d)
```

### 参数

- **d **-- 任何原生数据类型。

### 返回值

该方法返回 double 值。

# <a name="toRadians">toRadians()</a>

toRadians() 方法用于将角度转换为弧度。

### 语法

```java
double toRadians(double d)
```

### 参数

- **d **-- 任何原生数据类型。

### 返回值

该方法返回 double 值。

# <a name="random">random()</a>

random() 方法用于返回一个随机数，随机数范围为 0.0 =< Math.random < 1.0。

### 语法

```java
static double random()
```

### 参数

- 这是一个默认方法，不接受任何参数。

### 返回值

该方法返回 double 值。