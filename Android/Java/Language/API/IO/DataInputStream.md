# Java DataInputStream类

数据输入流允许应用程序以与机器无关方式从底层输入流中读取基本 Java 数据类型。

下面的构造方法用来创建数据输入流对象。

```java
DataInputStream dis = new DataInputStream(InputStream in);
```

另一种创建方式是接收一个字节数组，和两个整形变量 off、len，off表示第一个读取的字节，len表示读取字节的长度。

| 序号   | 方法描述                                     |
| ---- | ---------------------------------------- |
| 1    | **public final int read(byte[] r, int off, int len)throws IOException**从所包含的输入流中将 `len` 个字节读入一个字节数组中。如果len为-1，则返回已读字节数。 |
| 2    | **Public final int read(byte [] b)throws IOException**从所包含的输入流中读取一定数量的字节，并将它们存储到缓冲区数组 `b` 中。 |
| 3    | **public final Boolean readBooolean()throws IOException,****public final byte readByte()throws IOException,****public final short readShort()throws IOException****public final Int readInt()throws IOException**从输入流中读取字节，返回输入流中两个字节作为对应的基本数据类型返回值。 |
| 4    | **public String readLine() throws IOException**从输入流中读取下一文本行。 |

### 实例

下面的例子演示了DataInputStream和DataOutputStream的使用，该例从文本文件test.txt中读取5行，并转换成大写字母，最后保存在另一个文件test1.txt中。

```java
import java.io.*;

public class Test{
   public static void main(String args[])throws IOException{

      DataInputStream d = new DataInputStream(new
                               FileInputStream("test.txt"));

      DataOutputStream out = new DataOutputStream(new
                               FileOutputStream("test1.txt"));

      String count;
      while((count = d.readLine()) != null){
          String u = count.toUpperCase();
          System.out.println(u);
          out.writeBytes(u + "  ,");
      }
      d.close();
      out.close();
   }
}
```

以上实例编译运行结果如下：

```
THIS IS TEST 1  ,
THIS IS TEST 2  ,
THIS IS TEST 3  ,
THIS IS TEST 4  ,
THIS IS TEST 5  ,
```