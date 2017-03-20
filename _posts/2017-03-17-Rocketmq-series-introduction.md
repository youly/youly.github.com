---
layout: post
title: Rocketmq系列-概念
category: java
tag: [java, rocketmq]
---

### rocketmq 核心概念

> Apache RocketMQ™ is an open source distributed messaging and streaming data platform.

RocketMQ 是阿里开源的分布式消息系统，提供低延时、高可靠的消息发布与订阅服务。下图来自其官网介绍：

 ![rocketmq 模型](/assets/images/rmq-model.png)

1、message：信息的载体，rocketmq 里的消息包含 body，tag， properties。

2、topic: 生产者往其中投递消息，消费者从中消费消息，生产与消费必须指定topic。

3、group：一个消息的 topic 可以被多个组的生产者(producer group)生产，也可以被多个组的消费者(consumer group)消费。

4、queue：topic 的内部组成，存储消息的最小单元。一个 topic 通常有一个或者多个内部 queue。

5、offset：消息在 queue 里的偏移量。消费者消费消息时，broker会存储消费者的消费偏移量。

### rocketmq 部署模型

#### 物理部署结构

 ![rocketmq 物理模型](/assets/images/rmq-deployment-model.png)

1、NameServer集群：提供topic的路由信息，比如生产者发布消息前，可以从 nameserver 查询消息应该被发往哪个 broker。nameserver是无状态的，集群节点之间互不通信。

2、broker集群：从生产者接收消息并存储，后续消费者从这里消费消息。broker 分为 Master 与 Slave，一个 Master 可以对应多个 Slave，一个 Slave 只能对应一个 Master。broker 启动后，会定时将 topic 信息注册到所有的 nameserver。

3、producer集群：生产集群，与 nameserver 中的一台建立长连接，定时拉取 topic 信息。

4、consumer集群：消费集群，与 nameserver 中的一台建立长连接，定时拉取 topic 信息。

### rocketmq 提供了哪些亮点功能？

1、消息堆积，容量依磁盘大小而定

2、消息延迟发送，提供了固定的几个间隔的延迟时间设置

3、消息过滤，基于tag

4、消息的可靠存储

### 参考

1、[RocketMQ 原理简介](http://alibaba.github.io/RocketMQ-docs/document/design/RocketMQ_design.pdf)

2、[分布式开放消息系统(RocketMQ)的原理与实践](http://www.jianshu.com/p/453c6e7ff81c)

3、[Gitbook: RocketMQ原理详解](https://kgyhkgyh.gitbooks.io/rocketmq/content/index.html)

3、[RocketMQ core concept](https://rocketmq.incubator.apache.org/docs/core-concept/)
