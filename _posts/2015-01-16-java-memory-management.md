---
layout: post
title: Java内存管理
category: java
tags: [java]
---

###什么是内存管理

内存管理包括内存的分配和回收。

在C++语言中内存管理是由程序员显式地来完成的，这通常导致一些常见的bug，如悬空指针引用和垃圾对象导致的内存泄露。java语言自从java 2开始就提供了自动内存管理的功能：垃圾回收器。

###运行时内存组成

大致布局如下图（[来自](http://www.pointsoftware.ch/de/under-the-hood-runtime-data-areas-javas-memory-model/)）：

![类存布局](/assets/images/java_memory_layout.png)

堆内存：所有线程共享，存储 new Objects，<em>对象信息</em>

非堆内存，除了堆以外的内存，包括：

* 方法区：和堆一样，所有线程共享，存储加载的 <em>类</em> 描述信息（版本、字段、方法、接口等）、常量池、静态变量、即时编译的代码等数据。
* 虚拟机栈：线程私有，存储局部变量表、操作数栈、动态链接、方法出口等信息。局部变量表存放了编译期可知的各种基本数据类型（ boolean、 byte、 char、 short、 int、 float、 long、 double）、 对象引用。
* 本地方法栈：本地方法栈则为虚拟机使用到的Native方法服务。
* 程序计数器：线程私有，记录每个线程当前执行字节码指令地址。

具体布局如下图：
![类存布局](/assets/images/java_memory_layout_2.png)

###堆内存管理
从垃圾的回收的角度来看，堆内存可以分为两个区域：

新生代：eden + from supvivor space

年老代：新生代中经过几次垃圾回收仍存活的对象 + eden中存放不下的大对象。


###垃圾回收器应该满足的特性
1、正确性：有效数据保证永远不会被错误回收，垃圾数据不应该经过几轮垃圾回收后仍保留。

2、高效性：在保证应用程序不中断的情况下快速的回收垃圾。时间、空间、频次上达到一个平衡。

3、紧凑性：垃圾回收后不会产生过多的分片。

4、可扩展性：垃圾回收器不应该成为一个多核多线程应用程序中得一个性能瓶颈。

###分代回收
新生代：内存小，收集频繁，要求时间利用率高

老年代：内存大，频次低，要求空间利用率高

###JVM启动参数设置

-Xms，-Xmx：jvm初始化内存大小和最大内存大小，等于 年轻代大小 + 年老代大小 + 永久代大小 + 栈大小

-Xmn，-XX:NewSize：堆内存中年轻代内存大小。

-XX:MaxNewSize：堆内存中年轻代内存最大值。

-XX:MaxPermSize：永久代内存最大值。

-Xss：每个线程的堆栈大小。相同物理内存下，减小这个值能生成更多的线程。但是操作系统对一个进程内的线程数还是有限制的，不能无限生成，经验值在3000~5000左右。

-XX:NewRatio：Ratio of old/new generation sizes. The default value is 2.

XX:SurvivorRatio：Ratio of eden/survivor space size. The default value is 8.

###参考
1、[Memory Management in the Java HotSpot™ Virtual Machine](http://www.oracle.com/technetwork/java/javase/memorymanagement-whitepaper-150215.pdf)

2、[The Structure of the Java Virtual Machine](http://docs.oracle.com/javase/specs/jvms/se7/html/jvms-2.html)

3、[Configuring the Default JVM and Java Arguments](http://docs.oracle.com/cd/E22289_01/html/821-1274/configuring-the-default-jvm-and-java-arguments.html)

4、[Java HotSpot VM Options](http://www.oracle.com/technetwork/java/javase/tech/vmoptions-jsp-140102.html)
