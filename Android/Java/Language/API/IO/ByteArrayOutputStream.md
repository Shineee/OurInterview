# Java ByteArrayOutputStream类

字节数组输出流在内存中创建一个字节数组缓冲区，所有发送到输出流的数据保存在该字节数组缓冲区中。创建字节数组输出流对象有以下几种方式。

下面的构造方法创建一个32字节（默认大小）的缓冲区。

```java
OutputStream bOut = new ByteArrayOutputStream();
```

另一个构造方法创建一个大小为n字节的缓冲区。

```java
OutputStream bOut = new ByteArrayOutputStream(int a)
```

成功创建字节数组输出流对象后，可以参见以下列表中的方法，对流进行写操作或其他操作。

| 序号   | 方法描述                                     |
| ---- | ---------------------------------------- |
| 1    | **public void reset()**将此字节数组输出流的 `count` 字段重置为零，从而丢弃输出流中目前已累积的所有数据输出。 |
| 2    | **public byte[] toByteArray()**创建一个新分配的字节数组。数组的大小和当前输出流的大小，内容是当前输出流的拷贝。 |
| 3    | **public String toString()**将缓冲区的内容转换为字符串，根据平台的默认字符编码将字节转换成字符。 |
| 4    | **public void write(int w)** 将指定的字节写入此字节数组输出流。 |
| 5    | **public void write(byte []b, int of, int len)** 将指定字节数组中从偏移量 `off` 开始的 `len` 个字节写入此字节数组输出流。 |
| 6    | **public void writeTo(OutputStream outSt)**将此字节数组输出流的全部内容写入到指定的输出流参数中。 |

### 实例

下面的例子演示了ByteArrayInputStream 和 ByteArrayOutputStream的使用：

```java
import java.io.*;
public class ByteStreamTest {
   public static void main(String args[])throws IOException {
      ByteArrayOutputStream bOutput = new ByteArrayOutputStream(12);
      while( bOutput.size()!= 10 ) {
         // 获取用户输入
         bOutput.write(System.in.read()); 
      }
      byte b [] = bOutput.toByteArray();
      System.out.println("Print the content");
      for(int x= 0 ; x < b.length; x++) {
         // 打印字符
         System.out.print((char)b[x]  + "   "); 
      }
      System.out.println("   ");
      int c;
      ByteArrayOutputStream bInput = new ByteArrayOutputStream(b);
      System.out.println("Converting characters to Upper case " );
      for(int y = 0 ; y < 1; y++ ) {
         while(( c= bInput.read())!= -1) {
            System.out.println(Character.toUpperCase((char)c));
         }
         bInput.reset(); 
      }
   }
}
```

以上实例编译运行结果如下：

```
asdfghjkly
Print the content
a   s   d   f   g   h   j   k   l   y
Converting characters to Upper case
A
S
D
F
G
H
J
K
L
Y
```