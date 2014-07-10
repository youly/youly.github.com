---
layout: post
title: thrift简单使用
category: tools
tags: [thrift, tools]
---

###什么是thrift
[官网](http://thrift.apache.org/)上介绍：

>The Apache Thrift software framework, for scalable cross-language services development, combines a software stack with a code generation engine to build services that work efficiently and seamlessly between C++, Java, Python, PHP, Ruby, Erlang, Perl, Haskell, C#, Cocoa, JavaScript, Node.js, Smalltalk, OCaml and Delphi and other languages.

通常当网站达到一定规模时，单种编程语言已经无法满足最佳实践了。由于业务的需要，不同模块之间需建立一个简单稳定的数据交换通道。[SOAP](http://en.wikipedia.org/wiki/SOAP)是Web Services之间的交换结构化信息的一种协议，类似地thrift是实现本地程序之间交换数据的一种接口定义。

###thirft网络架构

[来自这里](http://thrift.apache.org/docs/concepts/):

     +-------------------------------------------+
      | Server                                    |
      | (single-threaded, event-driven etc)       |
      +-------------------------------------------+
      | Processor                                 |
      | (compiler generated)                      |
      +-------------------------------------------+
      | Protocol                                  |
      | (JSON, compact etc)                       |
      +-------------------------------------------+
      | Transport                                 |
      | (raw TCP, HTTP etc)                       |
      +-------------------------------------------+

Transport：负责读写数据，不管数据来源于套接字、内存还是文件。

Protocol：负责解析数据，client按此种格式encode，server端按此种格式decode。

Processor：负责执行远过程调用。

Server：具体的程序逻辑，负责创建Transport、Protocol、Processor，等待客户端连接并响应。

###参考
1、[Thrift: The Missing Guide](http://diwakergupta.github.io/thrift-missing-guide/#_versioning_compatibility)

2、[Thrift: Scalable Cross-Language Services Implementation](http://thrift.apache.org/static/files/thrift-20070401.pdf)





