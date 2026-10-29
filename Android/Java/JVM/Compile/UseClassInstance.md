# 使用类实例 

Java 虚拟机类实例通过 Java 虚拟机 new 指令创建。之前提到过，在 Java 虚拟机层面， 构造函数将会以一个编译器提供的以<init>命名的方法出现。这个特殊的名字的方法也被称作实例初始化方法（ §2.9）。一个类可以有多个构造函数，对应地也就会有多个实例初始化方法。一旦类实例被创建，那么这个实例包含的所有实例变量，除了在本身定义的以及父类中所定义的，都将被赋予默认初始值， 接着，新对象的实例初始化方法将会被调用。譬如下面示例：

```java
Object create() {

return new Object();

}
```

编译后代码如下：

```java
Method java.lang.Object create()

0 new #1 // Class java.lang.Object

3 dup

4 invokespecial #4 // Method java.lang.Object.<init>()V \
  
7 areturn
```

在参数传递和方法返回时， 类实例（作为 reference 类型）与普通的数值类型没有太大区别，reference 类型也有它自己类型专有的 Java 虚拟机指令，譬如：

```java
int i; // An instance variable

MyObj example() {

MyObj o = new MyObj();

return silly(o);

}

MyObj silly(MyObj o) {

if (o != null) {

return o;

} else {

return o;

}

}
```

编译后代码如下：

```java
Method MyObj example()

0 new #2 // Class MyObj

3 dup

4 invokespecial #5 // Method MyObj.<init>()V

7 astore_1

8 aload_0

9 aload_1

10 invokevirtual #4

// Method Example.silly(LMyObj;)LMyObj;

13 areturn

Method MyObj silly(MyObj)

0 aload_1

1 ifnull 6

4 aload_1

5 areturn

6 aload_1

7 areturn
```

类实例的字段（实例变量）将使用 getfield 和 putfield 指令进行访问，假设 i 是一个int 型的实例变量，且方法 getIt()和 setIt()的定义如下：

```java
void setIt(int value) {

i = value;

}

int getIt() {

return i;

}
```

编译后代码如下：

```java
Method void setIt(int)

0 aload_0

1 iload_1

2 putfield #4 // Field Example.i I

5 return

Method int getIt()

0 aload_0

1 getfield #4 // Field Example.i I

4 ireturn
```

无论方法调用指令的操作数，还是 putfield、 getfield 指令的操作数（上面例子中运行时常量池索引#4）都并非类实例中的地址偏移量。编译器会为这些字段生成符号引用，保存在运行时常量池之中。这些运行时常量池项会在解析阶段转换为引用对象中真实的字段位置。 















