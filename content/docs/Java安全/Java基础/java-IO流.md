---
title: Java IO流
date: 2026-04-29
tags: 
  - java安全
  - java-io流
categories: 
  - Web安全
  - java安全 
excerpt: java-io流初探
---

# Java IO流

## 流简介

### 什么是流

在 java 中，流是对数据传输的抽象。我们可以想象成一个水管，数据是水，从源流向目的地。不管数据来自文件、网络、内存还是键盘，处理方式都是类似的。

流的本质是**字节序列**，java 用统一的视角看待所有 I/O 操作。

IO 是指 Input/Output，即输入和输出，以内存为中心。

那么为什么要把数据读到内存中才能处理这些数据？因为代码都是在内存中运行的，数据也必须读取到内存中，最终的表示方式一般是 byte 数组，字符串等，都必须存放在内存中。

从 java 代码看，输入则是从外部（如硬盘上的某个文件）把内容读取到内存中，并且以 java 提供的某种数据类型表示。例如，`byte[]`、`String`等，这样后续代码才能处理这些数据。

因为内存有“易失性”的特点，即一断电数据就消失了，所以必须把处理之后的数据以某种形式输出，例如写入文件。Output 实际上就是把 java 表示的数据格式（如`byte[]`，`String`）输出到某个地方。

IO 流是一种顺序读写数据的模式，特点是单向流动，数据类似水一样。

## 创建文件的三种方式

### 根据路径创建一个 File 对象

使用`new File(String pathname)`

```java
package IOStream;

import java.io.File;
import java.io.IOException;

public class IOStreamTest {

    public static void createFile() throws Exception{
        File file = new File("C:\\Users\\32202\\Desktop\\java测试\\test\\src\\IOStream\\Files\\test.txt");
        try {
            file.createNewFile();
            System.out.println("successful");
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }

    public static void main(String[] args) throws Exception{
        createFile();
    }
}
```

### 根据父目录 File 对象，在子路径创建一个文件

使用`new File(File parent,String child)`

```java
package IOStream;

import java.io.File;
import java.io.IOException;

public class IOStreamTest {

    public static void createFile() throws Exception{
        File pfile = new File("C:\\Users\\32202\\Desktop\\java测试\\test\\src\\IOStream\\Files");
        File cfile = new File(pfile, "test1.txt");
        try {
            cfile.createNewFile();
            System.out.println("successful");
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }

    public static void main(String[] args) throws Exception{
        createFile();
    }
}
```

### 根据父目录路径，在子路径生成文件

使用`new File(String parent,String child)`

```java
package IOStream;

import java.io.File;
import java.io.IOException;

public class IOStreamTest {

    public static void createFile() throws Exception{
        String parentpath="C:\\Users\\32202\\Desktop\\java测试\\test\\src\\IOStream\\Files";
        String filename="test2.txt";
        File file = new File(parentpath, filename);
        try {
            file.createNewFile();
            System.out.println("successful");
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }

    public static void main(String[] args) throws Exception{
        createFile();
    }
}
```

## 获取文件信息

利用`File`类的一些方法获取基本信息：

```java
package IOStream;

import java.io.File;

public class GetFileInfo {
    public static void main(String[] args) {
        getFileContents();
    }

    public static void getFileContents(){
        File file = new File("C:\\Users\\32202\\Desktop\\java测试\\test\\src\\IOStream\\Files\\test.txt");
        System.out.println("文件名："+file.getName());
        System.out.println("文件绝对路径："+file.getAbsolutePath());
        System.out.println("文件上一级目录："+file.getParent());
        System.out.println("文件大小（字节）："+file.length());
        System.out.println("是不是文件："+file.isFile());
        System.out.println("是不是目录："+file.isDirectory());
    }
}
```

## 目录与文件操作

### 文件删除

```java
package IOStream;

import java.io.File;

public class FileDelete {

    public static void deleteFile(){
        File file = new File("C:\\Users\\32202\\Desktop\\java测试\\test\\src\\IOStream\\Files\\test2.txt");

        System.out.println(file.delete() ? "successful" : "failed");
    }
    public static void main(String[] args) {
        deleteFile();;
    }
}
```

### 目录删除

注：只有空的目录可以删除，不然会显示删除失败。

```java
package IOStream;

import java.io.File;

public class DeleteDict {
    public static void main(String[] args) {
        deleteDict();
    }

    public static void deleteDict(){
        File file = new File("C:\\Users\\32202\\Desktop\\java测试\\test\\src\\IOStream\\FileDeleteTest");
        System.out.println(file.delete()?"successful":"failed");
    }
}
```

### 创建单级目录

使用`file.mkdir`

```java
package IOStream;

import java.io.File;

public class CreateSingleDict {
    public static void main(String[] args) {
        create_single_dict();
    }

    public static void create_single_dict(){
        File file = new File("C:\\Users\\32202\\Desktop\\java测试\\test\\src\\IOStream\\CreateDict");
        System.out.println(file.mkdir()?"successful":"failed");
    }
}
```

### 创建多级目录

使用`file.mkdirs()`

```java
package IOStream;

import java.io.File;

public class CreateSingleDict {
    public static void main(String[] args) {
        create_single_dict();
    }

    public static void create_single_dict(){
        File file = new File("C:\\Users\\32202\\Desktop\\java测试\\test\\src\\IOStream\\CreateMultiDict\\test");
        System.out.println(file.mkdirs()?"successful":"failed");
    }
}
```

## IO流分类

按照处理单位分为：字节流和字符流

| 类型   | 处理单位                  | 基类                       | 适用场景                                 |
| ------ | ------------------------- | -------------------------- | ---------------------------------------- |
| 字节流 | 8位字节                   | `InputStream/OutputStream` | 图片、视频、音频、可执行文件等二进制数据 |
| 字符流 | 16位字符(在Unicode编码下) | `Reader/Writer`            | 文本文件、字符串等字符数据               |

按照数据流流向不同分为：输入流和输出流

按照流的角色不同分为：节点流、处理流/包装流

| 类型   | 说明                                                      |
| ------ | --------------------------------------------------------- |
| 节点流 | 直接与数据源/目标连接，如`FileInputStream`                |
| 处理流 | 包装节点流，添加缓冲、转换等功能，如`BufferedInputStream` |

## 文件流的一些操作

### Runtime 命令执行操作的 payload

```java
package CmdExec;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.io.InputStream;

public class RuntimeExec {
    public static void main(String[] args) throws IOException {
        InputStream inst = Runtime.getRuntime().exec("whoami").getInputStream();
        byte[] bytes = new byte[1024];
        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
        int readlen=0;
        while ((readlen=inst.read(bytes))!=-1){
            byteArrayOutputStream.write(bytes,0,readlen);
        }
        System.out.println(byteArrayOutputStream);
    }
}
```

这个 payload 把 IO 流和命令执行结合，本质是一个通过 IO 流窃取命令执行结果。

`InputStream inst = Runtime.getRuntime().exec("whoami").getInputStream();`这段代码建立了一个管道，主要干了三件事：

1. `exec("whoami")`：启动一个子进程来执行系统命令。
2. `getInputStream()`：获取子进程的标准输出流。

- 对于我们的 java 程序来说，获取子进程产生的数据是一个输入流
- 可以理解为，java 程序与子进程之间有一个管道相连，子进程把执行结果写入管道，我通过字节输入流从管道中读取出来。

```java
byte[] bytes = new byte[1024];
ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
```

这里的`byte[] bytes = new byte[1024]`是用于缓存数据的，是一个经典的中转处理流。`bytes`是一个缓冲区，每次从输入流中最多获取1024字节的数据。

`ByteArrayOutputStream()`：这是一个目的地在内存的字节输出流，会把所有从输入流中读取的数据暂时存到内存中。

```java
int readlen=0;
while ((readlen=inst.read(bytes))!=-1){
    byteArrayOutputStream.write(bytes,0,readlen);
}
```

这是 IO 流操作中最核心、最机械的循环模式。

1. `inst.read(bytes)`：从输入流中读取数据，放进`bytes`这个缓冲区，返回实际读取到的字节数。
2. `=-1`判断：如果读到末尾（子进程输出完毕，管道关闭），`read`方法会返回-1，循环结束。
3. `write`写入内存：把读取到的有效数据写入`ByteArrayOutputStream`内存仓库中。

循环结束，说明子进程`whoami`的所有输出已经转移到 JVM 内存中了。

```java
System.out.println(byteArrayOutputStream);
```

这里使用`println`会自动调用`byteArrayOutputStream.toString()`方法，使用系统默认的字符编码，把内存中的字节数组解码成字符串输出。

### FileInputStream

#### read() 方法

这个方法会从输入流中读取一个数据字节，如果没有输入，此方法将阻塞。返回值是下一个数据字节，如果已经到达文件末尾，则返回-1。

下面的样例使用`FileInputStream.read()`读取文件内容：

```java
package IOStream;

import java.io.FileInputStream;
import java.io.FileNotFoundException;
import java.io.IOException;

public class FileInputRead {
    public static void readfile() throws IOException {
        String pathname = "C:\\Users\\32202\\Desktop\\java测试\\test\\src\\IOStream\\Files\\test.txt";
        int readdata = 0;
        FileInputStream fileInputStream = null;
        try {
            fileInputStream = new FileInputStream(pathname);
            while ((readdata = fileInputStream.read()) != -1) {
                System.out.print((char) readdata);
            }
        } catch (FileNotFoundException e) {
            throw new RuntimeException(e);
        } catch (IOException e) {
            throw new RuntimeException(e);
        } finally {
            fileInputStream.close();
        }

    }

    public static void main(String[] args) throws IOException {
        readfile();
    }
}
```

`fileInputStream = new FileInputStream(pathname);`：在 java 程序和硬盘之间建立了一个字节输入管道。`FileInputStream`是节点流，直接连接数据源。

`while ((readdata = fileInputStream.read()) != -1)`：逐字节读取数据。因为流本质是二进制数据，所以我们接收数据的变量的数据类型要是数字类型的。**注：java 在读取文件中的字节时是按照无符号方式解读，所以一个字节的值的范围是从0~255，而 byte 类型数据是-128~127，已经超出范围，所以需要用 int 来接收数据。**

`System.out.print((char) readdata);`：这里则是把接收的数据经过 ASCII 编码强转成字符。

#### read(byte[] b) 方法

这个方法和上面的 read() 方法完全不同，下面是不同之处：

| 方法           | 一次读取量   | 返回值含义       |
| -------------- | ------------ | ---------------- |
| `read()`       | 1个字节      | 字节本身         |
| `read(byte[])` | 最多填满数组 | 实际读取的字节数 |

样例：

```java
package IOStream;

import java.io.FileInputStream;
import java.io.FileNotFoundException;
import java.io.IOException;

public class FileInputRead {
    public static void readfile() throws IOException {
        String pathname = "C:\\Users\\32202\\Desktop\\java测试\\test\\src\\IOStream\\Files\\test.txt";
        int readdata = 0;

        byte[] bytes = new byte[8];
        try (FileInputStream fileInputStream = new FileInputStream(pathname)) {
            while ((readdata = fileInputStream.read(bytes)) != -1) {
                System.out.println(new String(bytes, 0, readdata));
            }
        } catch (IOException e) {
            throw new RuntimeException(e);
        }

    }

    public static void main(String[] args) throws IOException {
        readfile();
    }
}
```

### FileOutputStream

#### write(int b) 方法

这个方法很简单，和上面的 read() 是相反的。read() 是逐个字节读取，而 write(int b) 是把 b 的最低8位写入文件，高24位直接忽略。

#### write(byte[] b) 方法

这个方法是`FileOutputStream`最常用的写入方式之一。

与 write(int b) 写单个字节不同，这个方法一次性写入整个字节数组到文件输出流中，没有返回值。

```java
package IOStream;

import java.io.FileOutputStream;

public class FileOutputWrite {
    public static void main(String[] args) throws Exception {
        writeFile();
    }

    public static void writeFile() throws Exception{
        String filepath="C:\\Users\\32202\\Desktop\\java测试\\test\\src\\IOStream\\Files\\test.txt";
        try (FileOutputStream fileOutputStream = new FileOutputStream(filepath)) {
            String contents="solo";
            fileOutputStream.write(contents.getBytes());
//String类型的字符串可以使用getBytes()方法将字符串转换为byte数组
        }catch (Exception e){
            e.printStackTrace();
        }
    }
}
```

#### write(byte[] b,int off,int len)

指定 byte[] 数组中从偏移量 off 开始的 len 个字节写入文件输出流。

```java
package IOStream;

import java.io.FileOutputStream;
import java.nio.charset.StandardCharsets;

public class FileOutputWrite {
    public static void main(String[] args) throws Exception {
        writeFile();
    }

    public static void writeFile() throws Exception{
        String filepath="C:\\Users\\32202\\Desktop\\java测试\\test\\src\\IOStream\\Files\\test.txt";
        try (FileOutputStream fileOutputStream = new FileOutputStream(filepath)) {
            String contents="solowalker";
            fileOutputStream.write(contents.getBytes(StandardCharsets.UTF_8),0,contents.length());

        }catch (Exception e){
            e.printStackTrace();
        }
    }
}
```

#### 追加写入

如果写入的数据不想覆盖之前的数据，可以设置`FileOutputStream`的构造方法`append`参数为`true`

```java
package IOStream;

import java.io.FileOutputStream;
import java.nio.charset.StandardCharsets;

public class FileOutputWrite {
    public static void main(String[] args) throws Exception {
        writeFile();
    }

    public static void writeFile() throws Exception{
        String filepath="C:\\Users\\32202\\Desktop\\java测试\\test\\src\\IOStream\\Files\\test.txt";

        try (FileOutputStream fileOutputStream = new FileOutputStream(filepath,true)){

            String contents="solo";
            fileOutputStream.write(contents.getBytes(StandardCharsets.UTF_8),0,contents.length());

        }catch (Exception e){
            e.printStackTrace();
        }
    }
}
```

### 文件拷贝

把`FileInputStream`和`FileOutputStream`结合起来，原理就是先把文件的内容读取出来，然后写入新的文件。

```java
package IOStream;

import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.nio.charset.StandardCharsets;

public class FileOutputWrite {
    public static void main(String[] args) throws Exception {
        copyfile();
    }

    public static void copyfile() throws Exception{
        String originalfile="C:\\Users\\32202\\Desktop\\java测试\\test\\src\\IOStream\\Files\\test.txt";
        String cpyfile="C:\\Users\\32202\\Desktop\\java测试\\test\\src\\IOStream\\Files\\test1.txt";
        byte[] bytes = new byte[1024];
        int readlen=0;
        try (FileInputStream fileInputStream = new FileInputStream(originalfile);
             FileOutputStream fileOutputStream = new FileOutputStream(cpyfile)) {
            while ((readlen=fileInputStream.read(bytes))!=-1){
                fileOutputStream.write(bytes,0,readlen);
            }
        }catch (Exception e){
            e.printStackTrace();
        }

    }
}
```

### FileReader

这是 java io 中最常用的字符文件输入流，和`FileInputStream`很像，但是有本质区别。

|          | FileInputStream              | FileReader                                          |
| -------- | ---------------------------- | --------------------------------------------------- |
| 父类     | `InputStream`                | `Reader`                                            |
| 处理单位 | 字节（byte）                 | 字符（char）                                        |
| 适用场景 | 图片视频等任意二进制文件     | 纯文本文件（.txt，.java，.xml）                     |
| 读取中文 | 会把多字节字符拆解，出现乱码 | 自动按照字符读取，中文完整没有乱码                  |
| 底层原理 | 直接读取字节                 | 内部使用`InputStream`读取字节，再按照编码解码成字符 |

```java
package IOStream;

import java.io.FileReader;

public class FileReadPrint {
    public static void main(String[] args) {
        readfile();
    }

    public static void readfile(){
        String filepath="C:\\Users\\32202\\Desktop\\java测试\\test\\src\\IOStream\\Files\\test.txt";
        try(FileReader fileReader = new FileReader(filepath)){
            int readlen=0;
            char[] codes = new char[8];
            while ((readlen= fileReader.read(codes))!=-1){
                System.out.println(new String(codes,0,readlen));
            }
        }catch (Exception e){
            e.printStackTrace();
        }

    }
}
```

这段代码和`FileInputStream`的意思是类似的。

## 总结

花了两个晚上左右的时间来搞定这一篇博客和 java io 流的学习，对 io 流这种东西算是有了一个系统的理解，剩下的就要在题目中学习了，光学习了理论没有进入实战环境感觉还是有点虚的，等 java 安全基础这部分理论学完就会开始 java 基础开发阶段和 cc 链。希望学习 java 安全能够顺利进行下去。

向着大厂 offer 进军！！！























