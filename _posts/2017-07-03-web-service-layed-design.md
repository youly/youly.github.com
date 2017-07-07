---
layout: post
title: Web服务分层设计模型
category: 方法论
tag: [架构, 质量保障]
---

### 引言
如何写好代码，是每个程序员都应该去关注的话题。代码不仅仅是给机器执行的，更是给人看的。我们都希望自己的代码写的好看犹如你的外表，但面对复杂业务需求和多人协同时，如果没有一套基本的原则规范，很难确保不迷失方向。本文就如何写好代码的问题，试图整理出一套规范。

### 什么是分层设计
简单的理解就是，把系统的逻辑功能按照某些原则抽象出几个层次，层次内保持高类聚，层次间保持低耦合，整个系统达到概念清晰，边界清晰，最终功能稳定。

分层设计在软件架构中是比较常见的，比如OSI七层网络模型、计算机系统分层模型。

<img src="/assets/images/web-layer-4.png" alt="osi网络分层" style="width: 220px;"/>
<img src="/assets/images/web-layer-3.jpg" alt="计算机系统分层" style="width: 220px;"/>

同样的，分层设计这种思想也可以扩展到平时写的业务代码中。

### Web服务分层设计模型
在讨论Web服务分层设计模型前，有必要先了解下微服务的概念，毕竟分层设计模型属于其中的一部分。

	The microservice architectural style is an approach to developing a single application as a suite of small services, each running in its own process and communicating with lightweight mechanisms, often an HTTP resource API. These services are built around business capabilities and independently deployable by fully automated deployment machinery. There is a bare minimum of centralized management of these services, which may be written in different programming languages and use different data storage technologies.

上文是 [Martin Fowler](https://martinfowler.com/articles/microservices.html) 给 microservice 的一个解释。微服务在当下是非常流行的架构，很多公司采用，也有一些开源的框架实现。实施微服务架构有几个重要的原则：

* 服务化的组件，组件服务化，通过服务接口来暴露组件
* 按业务能力来组织划分服务
* 面向产品，而不是项目，团队应该为产品的整个生命周期负责。
* 基础设施自动化，即应用的构建、集成测试、发布自动化。
* 还有其他几条原则…

微服务涉及的概念比较多，需要自己去多多阅读文档。下图是一个简单的架构描述，我们写的业务代码通常都是在其中的左侧绿色service部分：

<img src="/assets/images/web-layer-1.jpg" alt="常见系统架构" style="width: 660px;"/>

针对其中的service层业务，结合实战中的经验，总结出以下分层模型：

<img src="/assets/images/web-layer-2.jpg" alt="常见系统架构" style="width: 660px;"/>

* Vo: view object
* Bo: business object
* Co: cache object
* Do: database object

上层可以调用下层，下层不可调用上层，同层之间调用看情况：

* Facade层：对外暴露的服务(dubbo)接口，调用不同的service层服务，处理vo2bo/bo2vo的数据转换，接口版本兼容处理。同层之间不可以调用。
* Service层：业务领域内复杂逻辑处理(cpu计算密集型)和业务之间的数据交换，调用Component层接口获取数据。同层之间可以调用，可以越过component层调用cache、dao读接口。
* Component层：相同业务域service公共逻辑下层到本层，调用cache/dao层提供的接口，最小业务处理单元，不可再拆分。相互之间不可以调用。一般业务比较简单时，可以先不实现component。业务变复杂后，根据情况把公共部分下层到component层。
* Cache层：cache数据读写，包括组合的数据对象cache和单纯的dao cache，只处理缓存相关的读写，调用dao层接口填充cache。同层之间不可以调用。
* Dao层：database层的数据读写，负责业务无关操作，如db更新，数据变更触发的消息推送。一个dao对应一张表。

### 关于成本
实施微服务架构带来的开发/运维成本本身就比单体应用高很多，再去做分层处理，开发成本就更高了。因此对于追求业务快速上线的，没必要以这种方式去实施，但对于基础中台服务，稳定性、可维护性要求较高，此规范可做参考。

### 参考链接
1、[Microservices](https://martinfowler.com/articles/microservices.html)
