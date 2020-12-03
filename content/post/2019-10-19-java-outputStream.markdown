---
layout: post
title:  JAVA IO输出流总结
date:   2019-10-19 13:32:20 +0300
description: You’ll find this post in your `_posts` directory. Go ahead and edit it and re-build the site to see your changes. # Add post description (optional)
img: post-2.jpg # Add image post (optional)
tags: [Blog, Java OutputStream Or Writer]
author: martin6699s # Add name author (optional)
---


## JAVA-  OutputStream\FileOutputStream\OutputStreamWriter\BufferWriter\FileWriter总结



介绍：

1. OutputStream: 

   OutputStream是字节输出流的所有类的超类，是个抽象类。输出流接收到字节流并将其发送到某个底层接收器。输出流各式各样，有文件输出流(FileOutputStream)，GZIP输出流(GZIPOutputStream)，对象输出流(ObjectOutputStream)。 

2. FileOutputStream: 

   FIleOutputStream是用于将数据写入到输出流File或者文件描述符，文件是否打开或被创建取决于底层操作系统。有些操作系统只允许一次只能打开一个文件来写入文件字节流FileOutputStream。在这种情况下，如果文件已被打开或不存在且无法创建文件(权限原因)，则此类的构造方法将报错FileNotFoundException异常。**FileOutputStream用于如图像数据的原始字节流写入**，`对于写入字符流，请考虑使用FileWriter`。

3. OutputStreamWriter:

   说OutputStreamWriter之前，先说下OutputStreamWriter的超类Writer，Writer抽象类目的是为了能写入字符流，提供抽象方法write给子类自行实现，并**继承了flushable接口(实现了flushable接口类在使用flush()方法时，会将缓存输出流写入到底层设备上，如File输出流写入到文件中，网络套接字输出流经TCP/IP发送到底层设备 再传输到指定地址)**。封装了操作OutputStream输出的一系列动作和优化。

   而OutputStreamWriter是封装了OutputStream子类的Writer类，是把字符流（也就是字符数组或字符串）转换为字节流写入到底层设备。可以指定字符集，或使用系统默认字符集。每次调用write()方法都会在给定字符上调用编码转换器（也就是把字符转换为指定字符集的编码）,并把生成的字节在写入底层输出流之前写入到缓存区中积累**（缓冲区大小默认8192字节）**。【注意： 传递给write()方法的字符不会缓冲，转换的字节会被缓冲 】

   > 为了最大的效率，请考虑在BufferedWriter中包装一个OutputStreamWriter，以避免频繁的转换器调用。 例如：
   >
   > ```
   >   Writer out
   >    = new BufferedWriter(new OutputStreamWriter(System.out)); 
   > ```

   这种多次调用转换器的情况 就个人理解，发生在多次调用write方法才会发生。



4. FileWriter

FileWriter为写入文件提供便利，是OutputStreamWriter的子类，FileWriter类构造器中使用FileOutputStream传递给父类OutputStreamWriter操作。可以理解为FileOutputStream文件输出流和OutputStreamWriter的组合。**缺点是使用系统字符集，不能自己指定字符集。**



5. BufferedWriter:

顾名思义 带缓冲区的Writer类，写文本到指定的字符输出流，缓存字符，以便有效地写入字符、数组、字符串。除非需要及时输出，否则建议使用BufferedWriter包装OutputStreamWriter或者FileWriter等类写入字符，**BufferedWriter提供的缓冲区默认8192字符（也可以自己指定）， 可以减少多次为输出流分配内存**。

```java
FileOutputStream outputStream=new FileOutputStream(file);
OutputStreamWriter writer=new OutputStreamWriter(outputStream,"GBK");
BufferedWriter bufferedWriter=new BufferedWriter(writer);
```

 BufferedWriter封装的是writer，也就是当buffer满了让别的writer去处理输出，而OutputStreamWriter封装的事数据流(OutputStream)，它是让数据流去处理输出 





**要理清OutputStream和OutputStreamWriter的关系，OutputStream可以理解为一种资源，例如水，而OutputStreamWriter可以理解为一直操作工具，例如水龙头。水龙头控制着水的流出。而BufferedWriter可以理解为有储水箱的水龙头。**

**stream和writer的不同，一个是二进制数据，一个是字符数据** 

**上述OutputStreamWriter和BufferWriter说的缓冲是指程序本身的缓冲，不是操作系统层面的缓冲（或者TCP发送缓冲区），调用flush方法是时将程序缓冲区数据写入到操作系统缓冲上。**

[深入解析Java中Flushable接口的flush方法](https://blog.csdn.net/zhangcarcard/article/details/51210811)

 [FileOutputStream,OutputStreamWriter, BufferedWriter为什么连用？](https://blog.csdn.net/HD243608836/article/details/87882100)