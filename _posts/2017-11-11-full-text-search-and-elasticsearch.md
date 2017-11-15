---
layout: post
title: 全文本搜索原理与Elasticsearch实践(二) 
category: 搜索
tag: [设计]
---

[上一篇文章](http://blog.lastww.com/2017/11/04/full-text-search-and-elasticsearch)简要地谈到了全文本搜索基本原理，这篇接下来介绍下当前比较受欢迎的开源搜索引擎elasticsearch以及如何正确使用它。

### elastcisearch 基本概念
> Elasticsearch is a distributed, RESTful search and analytics engine capable of solving a growing number of use cases. As the heart of the Elastic Stack, it centrally stores your data so you can discover the expected and uncover the unexpected.

除了搜索，它还是一个扩展性极好的数据分析引擎，帮你洞察意料之外、情理之中的商业远见。使用ES(elasticsearch简写，下同)过程中经常会遇到一些概念，为了方便理解，下面选取最常见到的几个介绍下：

* index，索引，文档数据存储逻辑上的一个定义，很多元数据基于index存储。
* type，索引类型，为避免建立过多的索引而引进的一个特殊字段，比如两个索引，各有五个字段，其中四个一样，那么就没必要建两个索引了，正确的做法是定义一个索引，两个类型。
* analyzer，分析器，文档索引及搜索时进行分词用，可以给每个文本字段定义一个analyzer。
* mappings，索引的scheme，类似于mysql的表定义，约定了索引会有哪些字段，每个字段的类型及使用哪种分析器进行分词处理。
* shard，索引分片，是ES集群扩容缩容的最小单位。ES进行实时索引时，会根据定义的routing（一般是id）把文档路由一个shard中进行存储。


### elasticsearch 与 lucene

ES的搜索功能基于apache开源的[lucene](https://lucene.apache.org/)组件构建。lucene负责底层的索引构建及读取，而ES在其上实现了工程上的高可用及弹性伸缩。从功能分层的角度来看，ES与lucene的关系，可以类比于mysql与cobar、memcached与twemproxy，都以代理的方式屏蔽底层可能会变更的逻辑，区别是一个是代码层面，一个是架构层面。ES的扩展性与高可用性主要体现在：

* 文档数据按照hash算法分区存储，通过分区的增减实现集群容量的弹性伸缩。一个分区可以是一个index，也可以是一个shard。shard与lucene内部定义的index是一一对应的关系，如下图：

![es分片2](/assets/images/es-shard2.png)

* 每个shard主分片可以按情况配置多个副本，这些副本与主分片分布在不同的机器，保证主分片不可用后，副分片能晋升为主分片继续提供服务，从而提高可用性。

If the shard grows too big, you have two options: upgrading the hardware to scale up vertically, or rebuilding the entire Elasticsearch index with more shards, to scale out horizontally to more machines of the same kind.

### 容量规划
ES提供了良好的伸缩性，针对将来业务的快速增长可以很方便的扩容，但要正确利用这个特性，需要了解其弹性伸缩的原理。上面提到，ES进行扩容的最小单位是shard，因此容量规划时，至少要划分两个shard，否则将来就无法水平扩容、只能重建索引了。

然而规划多少个shard才算合理？这个官方认为没有一个标准的答案，需要根据自己的数据模型、业务增长以及机器性能来判断。有一个原则是 start big and scale down，可以理解为开始时多定义一些分片，这些分片存储在少量节点上，后面按情况进行节点扩容，把一些分片挪到新增的节点上，如下示意图：

![es分片2](/assets/images/scaling-stories.svg)


比较严谨的做法是，部署一个和线上环境机器同等配置的测试集群，这个集群只配置一个shard，然后搞两组线程分别进行索引的读写（索引数据就是你的业务数据，索引读写就是你的业务逻辑），直到响应时间不可接受为止，这个时候就确定了一个分片能存储的最大记录数，然后以总记录数除以这个，得出最终需要的分片数量。

### 部署架构选择

ES提供了两种数据查询方式，基于http的restful api和基于socket的java api。针对这两种方式，比较常见的部署架构如下图：

![es部署架构](/assets/images/es-arch.png)


### 制定合理的mappings
制定合理的mappings是非常重要的，主要在两个方便：

* 对于mappings变更，ES内部会选举一个master来协调，先停止所有的索引写入，然后把变更同步到数据节点，最后再唤醒索引等待的线程，从这点来看，频繁变更mappings存在大的风险。
* 字段类型的定义，决定了最终是进行过滤查询还是匹配查询，明显过滤查询的性能高于匹配查询，因此不要让ES去做无谓的资源消耗。 

### 编写高效的执行语句

* 编写执行语句，首先要选对上下文。执行语句主要分为两种，即查询语句和过滤语句。查询语句包含在查询上下文中（{“query”:{}}），过滤语句包含在过滤上下文中（{"filter":{}}）。这两种上下文，在ES内部是截然不同的查询计划，性能相差很大。

* 尽量设置routing，ES搜索模式是query and fetch，意思是协调节点收到请求后，它先去查询所有的节点获取符合条件的数据，然后进行归并排序拿到符合条件的文档id，最后根据id去各节点拿数据。设置了routing，可以避免很多资源消耗。

* 禁止深分页排序查询，因为ES内部排序是用内存优先级队列实现，如limit 10000，50这种查询，必将消耗很多内存，严重可导致outOfMemory.

### 搜索结果聚合统计
有时除了对文档进行搜索，还有进行聚合分析的需求，如按地域划分使用某产品的人群等，这种类似于mysql的group by。进行聚合统计查询时也需要注意：

* 避免在文本字段上进行term聚合

### 提高算法排序相关性
搜索结果的相关性影响到用户对于产品的满意度，因此提高排序结果的相关性非常重要，有几种模式：

* 定义 mappings 时，增加一个分值字段，以离线的方式根据业务逻辑算出每条记录的权重，作为分值存储到文档中，搜索时使用QueryBuilders.functionScoreQuery自定义分值贡献逻辑
* 针对优先级比较高的字段，设置较高的boost，比如同时包含某关键之的有title和body两个字段，明显title里包含更具相关性，可以设置title具有较高的boost值。
* 使用 explain，分析ES内部算分逻辑，不断优化算分逻辑。

---------------------

#### 附加建议

1、如何进行scale

* Scale down the amount of data processed or the resources needed to perform the processing;
* Scale up the computing resources on a node, via parallel processing and faster memory/storage technologies;
* Scale out the computing to distributed nodes in a cluster/cloud or at the edge where the data resides.

2、分片策略

* 基于时间戳，使用索引模板
* 基于用户
* 基于特定领域维度的routing

### 参考
[1、found-sizing-elasticsearch](https://www.elastic.co/blog/found-sizing-elasticsearch)

[2、The ideal Elasticsearch index](https://bonsai.io/2016/01/11/ideal-elasticsearch-cluster)

[3、Visualizing Lucene's segment merges](http://blog.mikemccandless.com/2011/02/visualizing-lucenes-segment-merges.html)

[4、Performance Considerations for Elasticsearch Indexing](https://www.elastic.co/blog/performance-considerations-elasticsearch-indexing)
