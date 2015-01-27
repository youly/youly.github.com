---
layout: post
title: Jetty源码阅读 - server
category: 源码
tags: [jetty, java, nio]
---

本文将分析jetty组件中server组件的源码，基于jetty-9.3.0。

###什么是Server

Jetty由若干个组件组成，Server则是连接这些组件的桥梁，其在jetty整个架构中的位置如下：

![jetty-high-level-architecture](/assets/images/jetty-high-level-architecture.png)

###Server类图

![jetty server](/assets/images/jetty_server.jpg)

其中蓝色的线代表“类继承”，绿色实线代表“接口继承”，绿色虚线代表“接口实现”。

###Server源码分析

###参考

1、[Jetty Architecture](http://www.eclipse.org/jetty/documentation/current/architecture.html#basic-architecture)

