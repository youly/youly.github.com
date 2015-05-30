---
layout: post
title: 使用rabbitmq提供高可用性队列服务
category: 分布式
tags: [queue,rabbitme,ha]
---

###为什么使用队列

![queue](/assets/images/queue.jpeg)

一个复杂的业务系统，为了保证尽可能小的响应时间和高的吞吐率，必然关联到模块的解耦和业务的拆分。非核心业务异步化，保证业务的关键路径尽可能短。而解耦和异步化的一个实现方式就是消息中间件。通过消息来连接不同的业务，在扩展性，系统稳定性上提供了一个很好的解决方案。

###为什么是rabbitmq

1、支持[AMQP](http://en.wikipedia.org/wiki/Advanced_Message_Queuing_Protocol)

2、消息可靠性：发送确认、接收回执、消息落地、高可用性

3、灵活的路由方式：点对点、广播，基于正则的转发

4、服务集群，节点间共享用户、队列、exchange等，节点间负载能够均衡

5、消息的高可用性：分布式的架构，以牺牲网络分区的代价，换来服务的高可用性和数据一致性

6、提供web的管理界面

7、提供多语言的客户端

###cap theorem

分布式服务架构需要参考的一个[重要定理](http://en.wikipedia.org/wiki/CAP_theorem)，在一个分布式的系统中，不可能同时达到以下三点：

1、一致性，所有节点在同一时刻保持着同样的状态

2、可用性，保证所有的请求都有一个响应

3、分区容忍，即使某个分区失败，还能持续提供服务

rabbitmq提供的高可用集群参考了这一理论，提供一致性的数据、高可用性的服务，但却牺牲了分区容忍。

这里提到cap theorem，只是了解，并未作深入研究。

###rabbitmq集群

由多个rabbitmq节点组成逻辑上的一个节点，每个节点间共享用户、vhost、队列信息、exchange等。除了队列的消息（只存储在创建此队列的节点上），其他所有对数据、状态的更新在每个节点中都执行了一遍。因此，无论连接集群中的那个节点，都能得到一致的结果。通常集群的部署方式如下：

![rabbitmq_ha](/assets/images/rabbitmq_ha.png)

实际上，rabbitmq的集群仅仅是提供了扩展性，能够在多节点间负载均衡，但消息仅仅是存储在其中一台节点，一旦这台节点挂掉，关联此队列的所有服务就不能用了。

###rabbitmq集群高可用性

通过设置policy，将队列消息镜像到所有节点，消息的发布与删除，都能同步到各个节点。这样，当master节点挂掉，可以从salve节点中再推举一台作为master。

针对此点，在运维rabbitmq集群过程中，需要注意到以下几点：

1、last shutdown, first startup

2、加入一个新的节点（salve），尽量不要显示地同步队列数据，那样会中断此队列的服务，让salve自然同步

3、不要将集群节点部署到跨网段里。

4、重启一个salve节点，其持久保存的数据将被丢弃。

5、需要移除master节点或者选择一个新的master时，要执行 rabbitmqctl forget_cluster_node，然后才启动salves。

###参考
1、[Rabbitmq Clustering Guide](https://www.rabbitmq.com/clustering.html)

2、[Rabbitmq Highly Available Queues](http://www.rabbitmq.com/ha.html)







