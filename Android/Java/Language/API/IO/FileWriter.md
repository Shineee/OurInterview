# Java FileWriter类

FileWriter类从OutputStreamReader类继承而来。该类按字符向流中写入数据。可以通过以下几种构造方法创建需要的对象。

在给出 File 对象的情况下构造一个 FileWriter 对象。

```java
FileWriter(File file)
```

在给出 File 对象的情况下构造一个 FileWriter 对象。

```java
 FileWriter(File file, boolean append)
```

构造与某个文件描述符相关联的 FileWriter 对象。

```java
FileWriter(FileDescriptor fd)
```

在给出文件名的情况下构造 FileWriter 对象，它具有指示是否挂起写入数据的 boolean 值。

```java
FileWriter(String fileName, boolean append)
```

创建FileWriter对象成功后，可以参照以下列表里的方法操作文件。

| 序号   | 方法描述                                     |
| ---- | ---------------------------------------- |
| 1    | **public void write(int c) throws IOException**写入单个字符c。 |
| 2    | **public void write(char [] c, int offset, int len)**写入字符数组中开始为offset长度为len的某一部分。 |
| 3    | **public void write(String s, int offset, int len)**写入字符串中开始为offset长度为len的某一部分。 |

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
      //创建 FileReader 对象
      FileReader fr = new FileReader(file); 
      char [] a = new char[50];
      fr.read(a); // 从数组中读取内容
      for(char c : a)
          System.out.print(c); // 一个个打印字符
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