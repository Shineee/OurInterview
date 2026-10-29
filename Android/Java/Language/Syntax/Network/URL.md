# Java URL处理

URL（Uniform Resource Locator）中文名为统一资源定位符，有时也被俗称为网页地址。表示为互联网上的资源，如网页或者FTP地址。

本章节我们将介绍Java是如处理URL的。URL可以分为如下几个部分。

```java
protocol://host:port/path?query#ref
```

protocols(协议)可以是 HTTP, HTTPS, FTP, 和File。port 为端口号。path为文件路径及文件名。

HTTP协议的URL实例如下:

```java
http://www.w3cschool.cc/index.html?language=cn#j2se
```

以上URL实例并未指定端口，因为HTTP协议默认的端口号为80。

## URL 类方法

在java.net包中定义了URL类，该类用来处理有关URL的内容。对于URL类的创建和使用，下面分别进行介绍。

java.net.URL提供了丰富的URL构建方式，并可以通过java.net.URL来获取资源。

| 序号   | 方法描述                                     |
| ---- | ---------------------------------------- |
| 1    | **public URL(String protocol, String host, int port, String file) throws MalformedURLException.**通过给定的参数(协议、主机名、端口号、文件名)创建URL。 |
| 2    | **public URL(String protocol, String host, String file) throws MalformedURLException**使用指定的协议、主机名、文件名创建URL，端口使用协议的默认端口。 |
| 3    | **public URL(String url) throws MalformedURLException**通过给定的URL字符串创建URL |
| 4    | **public URL(URL context, String url) throws MalformedURLException**使用基地址和相对URL创建 |

URL类中包含了很多方法用于访问URL的各个部分，具体方法及描述如下：

| 序号   | 方法描述                                     |
| ---- | ---------------------------------------- |
| 1    | **public String getPath()**返回URL路径部分。    |
| 2    | **public String getQuery()**返回URL查询部分。   |
| 3    | **public String getAuthority()**获取此 URL 的授权部分。 |
| 4    | **public int getPort()**返回URL端口部分        |
| 5    | **public int getDefaultPort()**返回协议的默认端口号。 |
| 6    | **public String getProtocol()**返回URL的协议  |
| 7    | **public String getHost()**返回URL的主机      |
| 8    | **public String getFile()**返回URL文件名部分    |
| 9    | **public String getRef()**获取此 URL 的锚点（也称为"引用"）。 |
| 10   | **public URLConnection openConnection() throws IOException**打开一个URL连接，并运行客户端访问资源。 |

### 实例

以上实例演示了使用java.net的URL类获取URL的各个部分参数：

```java
// 文件名 : URLDemo.java

import java.net.*;
import java.io.*;

public class URLDemo
{
   public static void main(String [] args)
   {
      try
      {
         URL url = new URL("http://www.w3cschool.cc/index.html?language=cn#j2se");
         System.out.println("URL is " + url.toString());
         System.out.println("protocol is "
                                    + url.getProtocol());
         System.out.println("authority is "
                                    + url.getAuthority());
         System.out.println("file name is " + url.getFile());
         System.out.println("host is " + url.getHost());
         System.out.println("path is " + url.getPath());
         System.out.println("port is " + url.getPort());
         System.out.println("default port is "
                                   + url.getDefaultPort());
         System.out.println("query is " + url.getQuery());
         System.out.println("ref is " + url.getRef());
      }catch(IOException e)
      {
         e.printStackTrace();
      }
   }
}
```

以上实例编译运行结果如下:

```
URL is http://www.w3cschool.cc/index.html?language=cn#j2se
protocol is http
authority is www.w3cschool.cc
file name is /index.htm?language=cn
host is www.amrood.com
path is /index.html
port is -1
default port is 80
query is language=cn
ref is j2se
```

## URLConnections 类方法

openConnection() 返回一个 java.net.URLConnection。

例如：

- 如果你连接HTTP协议的URL, openConnection() 方法返回 HttpURLConnection 对象。
- 如果你连接的URL为一个 JAR 文件, openConnection() 方法将返回 JarURLConnection 对象。
- 等等...

URLConnection 方法列表如下：

| 序号   | 方法描述                                     |
| ---- | ---------------------------------------- |
| 1    | **Object getContent() **检索URL链接内容        |
| 2    | **Object getContent(Class[] classes) **检索URL链接内容 |
| 3    | **String getContentEncoding() **返回头部 content-encoding 字段值。 |
| 4    | **int getContentLength() **返回头部 content-length字段值 |
| 5    | **String getContentType() **返回头部 content-type 字段值 |
| 6    | **int getLastModified() **返回头部 last-modified 字段值。 |
| 7    | **long getExpiration() **返回头部 expires 字段值。 |
| 8    | **long getIfModifiedSince() **返回对象的 ifModifiedSince 字段值。 |
| 9    | **public void setDoInput(boolean input)**URL 连接可用于输入和/或输出。如果打算使用 URL 连接进行输入，则将 DoInput 标志设置为 true；如果不打算使用，则设置为 false。默认值为 true。 |
| 10   | **public void setDoOutput(boolean output)**URL 连接可用于输入和/或输出。如果打算使用 URL 连接进行输出，则将 DoOutput 标志设置为 true；如果不打算使用，则设置为 false。默认值为 false。 |
| 11   | **public InputStream getInputStream() throws IOException**返回URL的输入流，用于读取资源 |
| 12   | **public OutputStream getOutputStream() throws IOException**返回URL的输出流, 用于写入资源。 |
| 13   | **public URL getURL()**返回 URLConnection 对象连接的URL |

### 实例

以下实例中URL采用了HTTP 协议。 openConnection 返回HttpURLConnection对象。

```java
// 文件名 : URLConnDemo.java

import java.net.*;
import java.io.*;
public class URLConnDemo
{
   public static void main(String [] args)
   {
      try
      {
         URL url = new URL("http://www.w3cschool.cc");
         URLConnection urlConnection = url.openConnection();
         HttpURLConnection connection = null;
         if(urlConnection instanceof HttpURLConnection)
         {
            connection = (HttpURLConnection) urlConnection;
         }
         else
         {
            System.out.println("Please enter an HTTP URL.");
            return;
         }
         BufferedReader in = new BufferedReader(
         new InputStreamReader(connection.getInputStream()));
         String urlString = "";
         String current;
         while((current = in.readLine()) != null)
         {
            urlString += current;
         }
         System.out.println(urlString);
      }catch(IOException e)
      {
         e.printStackTrace();
      }
   }
}
```

以上实例编译运行结果如下：

```
$ java URLConnDemo

.....a complete HTML content of home page of amrood.com.....
```