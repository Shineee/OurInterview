# Java FileReader类

FileReader类从InputStreamReader类继承而来。该类按字符读取流中数据。可以通过以下几种构造方法创建需要的对象。

在给定从中读取数据的 File 的情况下创建一个新 FileReader。

```java
FileReader(File file)
```

在给定从中读取数据的 FileDescriptor 的情况下创建一个新 FileReader。

```java
FileReader(FileDescriptor fd) 
```

在给定从中读取数据的文件名的情况下创建一个新 FileReader。

```java
FileReader(String fileName) 
```

创建FIleReader对象成功后，可以参照以下列表里的方法操作文件。

| 序号   | 文件描述                                     |
| ---- | ---------------------------------------- |
| 1    | **public int read() throws IOException**读取单个字符，返回一个int型变量代表读取到的字符 |
| 2    | **public int read(char [] c, int offset, int len)**读取字符到c数组，返回读取到字符的个数 |

### 实例

```java
import java.io.*;
public class FileRead{
   public static void main(String args[])throws IOException{
      File file = new File("Hello1.txt");
      // 创建文件
      file.createNewFile();
      // creates a FileWriter Object
      FileWriter writer = new FileWriter(file); 
      // 向文件写入内容
      writer.write("This\n is\n an\n example\n"); 
      writer.flush();
      writer.close();
      // 创建 FileReader 对象
      FileReader fr = new FileReader(file); 
      char [] a = new char[50];
      fr.read(a); // 读取数组中的内容
      for(char c : a)
          System.out.print(c); // 一个一个打印字符
      fr.close();
   }
}
```

以上实例编译运行结果如下：

```
This
is
an
example
```