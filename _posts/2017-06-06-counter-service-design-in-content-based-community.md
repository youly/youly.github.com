---
layout: post
title: 内容社区产品中的计数服务设计
category: 设计
tags: [架构, 缓存]
---

### 引言
内容产品中比较常见的就是各种计数，比如点赞数、评论数、粉丝数等。由于近期需要实现计数的功能，因此针对如何计数实现作了下思考，并总结成本文。

### 目前想到的几种实现方案

#### 方案一、使用数据库提供的Count函数
简单粗暴，解决问题的最快方式。比较常见的使用场景有交易系统中的订单计数，如下图，以用户为维度按状态进行分组计数，更新不是很频繁，从性能上考虑，可以在上层做缓存。

<img src="/assets/images/counter1.png" alt="电商产品计数" style="width: 220px;"/>

然而最快并不一定意味着最好，在单用户记录数比较多的时候，利用数据库去做聚合统计成本还是相当高的，在列表场景下就更不适合了。

#### 方案二、独立的计数表
类似如下的表结构：

    +----------+--------+-------------+
    | biz_type | biz_id | biz_counter |
    +----------+--------+-------------+
    |        1 |   8094 |           3 |
    |        2 |   8095 |           9 |
    |        3 |   8099 |           7 |
    +----------+--------+-------------+
    
通过业务类型、业务id的唯一性来关联计数。业务数据的更新，同步更新db中的计数记录。同样从性能上考虑，可以在上层做一层缓存，计数更新，同步清除缓存。

<img src="/assets/images/counter2.png" alt="内容产品计数" style="width: 220px;"/>

相比方案一，会减少很多db的慢查询，也适合列表式的场景，但考虑业务发展到一定规模后，尤其是内容社交型的产品，数据频繁更新，tps瞬间峰值高，导致db的开销比较大，缓存命中率也不高，故此方案也只适用于中小型产品。

#### 方案三、Redis 内存存储

注意，标题是内存存储，不是内存缓存。内存存储是不设过期时间的。如果业务发展迅速，导致内存容量不够，像db扩容一样，内存也要进行扩容。

<img src="/assets/images/counter4.png" alt="社交产品计数" style="width: 220px;"/>

虽然是内存存储，但redis是支持数据落地的，因此不必担心数据丢失。访问计数不存在，上层业务可以认为计数值就是0。这一点就避免了大部分没有计数的数据（比如无赞无评论的问题）占用不必要的内存。另外redis设计本身支持原子性的增减操作，如命令HINCRBY、INCRBY，因此也不必担心数据被改错的问题。

只要保证redis的高可用，此方案相比方案一、二在性能上是最优的。然而好的东西总要付出代价，毕竟内存是昂贵的。在成本可控的情况下，要保证服务可用性，只能在软件设计上进行突破了。

### 折中的方案
方案三的性能最优，但受内存成本限制，因此需要在成本与用户体验上做出一个权衡。基本思想是：

* 使用近似内存存储，设置相对较长的过期时间，挺过热数据阶段
* 提高内存缓存的命中率，每次缓存更新，重置过期时间
* 增加独立的计数表，缓存失效，从计数表中读取数据，并同步更新缓存，尽量避免使用count函数
* 除了以上之外，可以干脆不设置过期时间，但当redis容量达到一定阈值后，需要人工介入，把冷数据批量清除，以释放内存存储。

基于以上思想，来看下面的计数服务架构图：

![计数服务架构](/assets/images/counter3.jpg)

说明：

* 计数器初始化，以select count(*) from table 的形式。项目上线前需要执行这一步
* 计数写入，先更新redis，然后通过队列异步更新db，保证数据落地
* 计数查询，优先查询redis，当redis挂掉或缓存过期，查CounterTable表，并异步初始化缓存
* 为保证数据正确性，上一步中的异步初始化缓存，通过步骤1中的方式进行初始化，并校验CounterTable数据正确性

需要额外注意的是，计数写入要进行判断：

* redis更新完成后，判断返回值，如果是1，则需要判断缓存是否过期，如果过期，则INCRBY/HINCRBY的结果是错误的，重新初始化缓存
* 或者写入之前先执行HEXISTS/EXISTS判断key是否存在，不存在则不写入，异步执行缓存初始化。

<b>以上都无法根本避免，在线程高并发的情况下，从取完初始化的数据到执行完HMSET期间数据丢失的可能。不过在缓存已经过期的情况下，业务上基本很少有高并发了</b>

欢迎有更好的方案提出来与大家共享。
