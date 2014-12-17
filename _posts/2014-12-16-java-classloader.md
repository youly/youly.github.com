---
layout: post
title: Java类加载器-ClassLoader
category: java
tags: [java, classloader]
---

###什么是ClassLoader
简单地将，ClassLoader是类加载器，运行时动态加载类定义。给定一个类名，ClassLoader尝试获取或者生成此类的定义。最常见的是从文件系统读取编译完成的class文件来加载类。

###什么时候需要ClassLoader
1、使用new方法或者Class.forName创建类的实例

2、使用类或接口的静态变量，而此类当前仍未加载

3、使用类的静态方法，而此类当前仍未加载

4、初始化一个类的子类

5、Java虚拟机启动时运行启动类main方法（Main Class）

###ClassLoader遵循的原则
1、ClassLoader以层次结构组织，除了根类加载器(bootstrap class loader)，每个classloader必须要有一个“父”classloader。虽然是层级关系，但不是实际上的继承关系，这么做主要是解决安全问题，把类以命名空间隔离开来。

示例图，来自[这里](http://javaeesupportpatterns.blogspot.com/2012/08/javalangnoclassdeffounderror-parent.html)：

![jvm_classloader](/assets/images/JVM_classloader_delagation_model_parent_first.png)

2、加载一个类时，类加载器应该先委托它的父类加载器来加载，层层委托，如果上层classloader都无法加载，则由自己来加载，自己无法加载，抛ClassNotFoundException。

3、一个class由类的名称和它的类加载器定义。

4、一个class仅被加载一次，加载之后字节码被缓存于它的classloader中。

5、class中的符号链接在此链接指向的类真正被使用时由jvm决定使用哪个classloader来加载，即这个classloader当前是不确定的。

6、类型转换时，如果源类和目标类对应的classloader不同，转换将失败。

###类文件结构
ClassLoader如何加载类，还需了解下类文件结构。

请参考这里：[http://docs.oracle.com/javase/specs/jvms/se7/html/jvms-4.html](http://docs.oracle.com/javase/specs/jvms/se7/html/jvms-4.html)

    ClassFile {
        u4             magic;
        u2             minor_version;
        u2             major_version;
        u2             constant_pool_count;
        cp_info        constant_pool[constant_pool_count-1];
        u2             access_flags;
        u2             this_class;
        u2             super_class;
        u2             interfaces_count;
        u2             interfaces[interfaces_count];
        u2             fields_count;
        field_info     fields[fields_count];
        u2             methods_count;
        method_info    methods[methods_count];
        u2             attributes_count;
        attribute_info attributes[attributes_count];
    }


###JDK ClassLoader
jdk自带三种的类加载器：

1、BootStrap ClassLoader：启动类加载器，这个所有类加载的根，负责加载JDK中的核心类库，类路径为sun.boot.class.path指定

2、Extension ClassLoader：扩展类加载器，加载jdk中得扩展类库，路径为JAVA_HOME/jre/lib/ext

3、App ClassLoader：应用程序类加载器，负责加载应用程序classpath目录下的所有jar和class文件

这三种类加载器的关系如下图：

![classloader_hierarchy](/assets/images/classloader_hierarchy.gif)

###自定义ClassLoader
为什么要自定义classloader呢？

有的时候需要隔离包之间的访问权限，比如一个系统使用java编写，它依赖一些外部包，而不想使用此系统的应用程序类感觉到这些外部包的存在，此系统可以实现自己的classloader。

还有些时候，我们会动态生成一些类，也需要用到自定义classloader。

实现自定义classloader很简单，jdk提供了一个抽象类ClassLoader，只需要重写它的findClass方法。

示例代码可点击[此处](https://github.com/youly/study/blob/master/src/main/java/com/lastww/study/basis/MyClassLoader.java)

###参考
1、[ClassLoader](http://docs.oracle.com/javase/7/docs/api/java/lang/ClassLoader.html)

2、[Understanding Network Class Loaders](http://www.oracle.com/technetwork/articles/javase/classloaders-140370.html)

3、[JVM 类的加载、连接和初始化 手动加载类](http://www.itzhai.com/java-virtual-machine-notes-jvm-class-loading-connection-and-manually-load-the-class-initialization.html)

4、[Inside Class Loaders](http://www.onjava.com/pub/a/onjava/2003/11/12/classloader.html)
