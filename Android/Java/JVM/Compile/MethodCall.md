# 方法调用 

对普通实例方法调用是在运行时根据对象类型进行分派的（相当于在 C++中所说的 “虚方法”）， 这类方法通过调用 invokevirtual 指令实现， 每条 invokevirtual 指令都会带有一个表示索引的参数，运行时常量池在该索引处的项为某个方法的符号引用，这个符号引用可以提供方法所在对象的类型的内部二进制名称、方法名称和方法描述符。下面这个例子中定义了一个实例方法 add12and13()来调用前面的 addTwo()方法，如下：

```java
int add12and13() {

return addTwo(12, 13);

}
```

编译后代码如下：

```java
Method int add12and13()

0 aload_0 // Push local variable 0 (this)

1 bipush 12 // Push int constant 12

3 bipush 13 // Push int constant 13

5 invokevirtual #4 // Method Example.addtwo(II)I

8 ireturn // Return int on top of operand stack; it is the int

result of addTwo()
```

方法调用过程的第一步是将当前实例的自身引用“ this” 压入到操作数栈中。传递给方法的参数， int 值 12 和 13 随后入栈。当调用 addTwo()方法时， Java 虚拟机会创建一个新的栈帧，传递给 addTwo()方法的参数作为新的帧的对应局部变量的初始值。即 this 和两个传递给的addTwo()方法的参数 12 和 13 被作为 addTwo()方法栈帧的第 0、 1、 2 个局部变量。
最后， 当 addTwo()方法执行结束、方法返回时， int 型的返回值被压入方法调用者的栈帧的操作数栈，即 add12and13()方法的操作数栈中。这样 addTwo()方法的返回值就被放置在调用者 add12and13()方法的代码可以立刻使用到的地方。
add12and13()方法的返回过程由 add12and13()方法中的 ireturn 指令实现。 由于调用的 addTwo()方法返回的 int 值被压入当前操作数栈的栈顶， ireturn 指令将会把当前操作数栈的栈顶值（此处就是 addTwo()的返回值） 压入调用 add12and13()方法的操作数栈。然后跳转至调用者调用方法的下一条指令继续执行， 并将调用者的栈帧重新设为当前栈帧。 Java 虚拟机对不同数据类型（ 包括声明为 void，没有返回值的方法）的返回值提供了不同的方法返回指令， 各种不同返回值类型的方法都使用这一组返回指令来返回。
invokevirtual 指令操作数（ 在上面示例中的运行时常量池索引#4） 不是 Class 实例中方法指令的偏移量。编译器并不需要了解 Class 实例的内部布局， 它只需要产生方法的符号引用并保存于运行时常量池即可， 这些运行时常量池项将会在执行时转换成调用方法的实际地址。 Java虚拟机指令集中其他指令在访问 Class 实例时也采用相同的方式。 

如果上个例子中，调用的实例方法 addTwo()变成类（ static） 方法的话，编译代码会有略微变化，如下：

```java
int add12and13() {

return addTwoStatic(12, 13);

}
```

编译代码中使用了另一个 Java 虚拟机调用指令 invokestatic：

```java
Method int add12and13()

0 bipush 12

2 bipush 13

4 invokestatic #3 // Method Example.addTwoStatic(II)I

7 ireturn
```

类方法和实例方法的调用的编译代码很类似， 两者的区别仅仅是实例方法需要调用者传递this 参数而类方法则不用。所以在两种方法的局部变量表中， 序号为 0（ 第一个） 的局部变量会有所区别（参见“接收参数”）。 invokestatic 指令用于调用类方法。
invokespecial 指令用于调用实例初始化方法（参见§3.8 “使用类实例”）， 它也可以用来调用父类方法和私有方法。譬如下面例子中的 Near 和 Far 两个类：

```java
class Near {

int it;

public int getItNear() {

return getIt();

}

private int getIt() {

return it;

}

}

class Far extends Near {

int getItFar() {

return super.getItNear();

}

}
```

调用类 Near 的方法 getItNear()（调用私有方法） 被编译为：

```java
Method int getItNear()

0 aload_0

1 invokespecial #5 // Method Near.getIt()I

4 ireturn 
```

调用类 Far 的方法 getItFar ()（调用父类方法）被编译为：

```java
Method int getItFar()

0 aload_0

1 invokespecial #4 // Method Near.getItNear()I

4 ireturn
```

请注意，所有使用 invokespecial 指令调用的方法都需要 this 作为第一个参数， 保存在第一个局部变量之中。
如果编译器要调用某个方法，必须先产生这个方法描述符，描述符中包含了方法实际参数和返回类型。编译器在方法调用时不会处理参数的类型转换问题，只是简单地将参数的压入操作数栈，且不改变其类型。通常，编译器会先把一个方法所在的对象的引用压入操作数栈，方法参数则按顺序跟随这个对象之后入栈。编译器在生成 invokevirtual 指令时，也会生成这条指令所引用的描述符，这个描述符提供了方法参数和返回值的信息。 作为方法解析时的一个特殊处理过程， 一个用于调用 java.lang.invoke.MethodHandle 的 invoke()或者
invokeExact()方法的 invokevirtual 指令会提供一个方法描述符，这个方法描述符符合语法规则，并且在描述符中确定的类型信息将会被解析。 











