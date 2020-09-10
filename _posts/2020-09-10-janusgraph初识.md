---
layout: post
title: janusGraph
date: 2020-09-08
tags: tools
---

## JanusGraph初识

## 目录

1. #### [简介](#简介)

2. #### [**索引**](#JanusGraph中的索引)

3. #### [事务](#JanusGraph中的事务)

4. #### [缓存](#JanusGraph中的缓存)

5. #### [存储](#JanusGraph数据存储)

---

> [原文参考](https://zhuanlan.zhihu.com/p/50279477)

### 简介

![image](https://user-images.githubusercontent.com/39090338/92711975-9d1abb80-f38b-11ea-9057-9a6b52f8c88d.png)

​		Apache TinkerPop是一款开源的图计算框架，JanusGraph通过原生支持TinkerPop来提供图数据库(OLTP)和图分析系统 (OLAP)的能力。下图包括了TinkerPop的几个组成部分。最上层是Gremlin Server，为用户提供了访问图数据库的通道，通过脚本可以连上Gremlin Server，使用Gremlin查询/遍历语言来操作JanusGraph等支持Gremlin的图数据库，Gremlin语言支持Cypher语法；再下层是TinkerPop提供的用于表示属性图的核心API，包括图、定点和边的各种操作接口；Graph Computer是TinkPop的图计算框架，与Spark、Giraph或Hadoop等大数据平台配合并通过调用JanusGraph提供的OLAP I/O接口实现图分析和处理能力。JanusGraph等图数据库和图分析系统通过实现Provider API来对接TinkPop框架，定制所需的功能。

![image](https://user-images.githubusercontent.com/39090338/92711990-a146d900-f38b-11ea-8600-619948b4e8d0.png)

### JanusGraph中的索引

**JanusGraph包括图索引（Graph Index）和顶点为中心的索引（Vertex-centric Indexes）**

图索引支持过滤索引，就是仅在满足特定条件的边或顶点建立索引。其又包括复合索引（Composite Index）和混合索引（Mixed Index）

**复合索引**只能进行等值的，包括所有索引键的查询。也就是说非等的查询，或者只包含部分键的查询都无法走复合索引。复合索引不依赖索引后端，其仅依赖存储后端，所以未配置索引后端不影响复合索引使用。order by需要在内存中排序，因为没有索引后端。可以通过unique()来限定某个复合索引是唯一索引。

**在deep上创建一个索引**

```text
gremlin> mgmt = graph.openManagement()
==>org.janusgraph.graphdb.database.management.ManagementSystem@151ab2b9
gremlin> deep = mgmt.getPropertyKey('deep')
==>deep
gremlin> mgmt.buildIndex('byDeepComposite', Vertex.class).addKey(deep).buildCompositeIndex()
==>byDeepComposite
gremlin> mgmt.commit()
==>null
```

**混合索引**类似于数据库常规索引，提供比复合索引更大的灵活性。其依赖索引后端，一个JanusGraph实例可以同时配置多个索引后端。混合索引支持全文索引，地理位置索引，范围查询等，默认创建的是全文索引类型。混合索引不具备唯一性。如果order by字段为索引的一部分，则可以下推到索引后端执行。在此不做展开。

对于已经有数据的边或顶点创建索引，在索引覆盖所有数据之前，索引不生效。可使用如下命令完成**索引重建**

```text
gremlin> mgmt = graph.openManagement()
==>org.janusgraph.graphdb.database.management.ManagementSystem@411cd156
gremlin> mgmt.updateIndex(mgmt.getGraphIndex("byDeepComposite"), SchemaAction.REINDEX).get()
==>org.janusgraph.diskstorage.keycolumnvalue.scan.StandardScanMetrics@5bcec67e
gremlin> mgmt.commit()
==>null
```

顶点中心索引，是局部索引，只为与顶点相连的某类边建立索引。其是前缀索引，所以键的先后顺序对索引效率有较大影响。仅支持等值，范围和包含类查询

```text
gremlin> mgmt = graph.openManagement()
==>org.janusgraph.graphdb.database.management.ManagementSystem@6c8f60f3
gremlin> time = mgmt.getPropertyKey('time')
==>time
gremlin> battled = mgmt.getEdgeLabel('battled')
==>battled
gremlin> mgmt.buildEdgeIndex(battled, 'battlesByTime1', Direction.BOTH, Order.decr, time)
==>battlesByTime1
gremlin> mgmt.commit()
==>null
```

### JanusGraph中的事务

JanusGraph中的事务是自动开启（与MySQL类似），但不会自动commit或rollback的（与MySQL不同），所有的读写操作都在事务中执行。形如：

```text
juno = graph.addVertex() //Automatically opens a new transaction
juno.property("name", "juno")
graph.tx().commit() //Commits transaction
```

所以，如果一直执行插入或更新，那么会导致隐式开启的事务越来越大。如果一个图数据库实例被close()时，还有事务在执行中，那么该事务的状态是未定的（TinkPop语义），对于隐式开启的与该实例线程绑定的事务，jg会将其提交掉，如果是显式开启的非线程绑定的事务，则会被回滚掉。

JanusGraph采用乐观事务模型，在提交时才会进行冲突检测，那么会导致在一个长事务中较早插入的具有唯一性约束的记录，在事务执行过程中被其他事务提交了。针对这种情况，可以在一个事务里面开启子事务，让子事务执行插入操作并先提交，这样能够提交长事务执行效率，避免无谓的失败。

多个事务间的隔离。JanusGraph中的事务类似于可重复读级别。事务开启后就会获取系统的状态（Snapshot？），在提交前不会更新，那么如果其他事务做了新的操作并提交，该事务是无法看到这些操作的。

### JanusGraph中的缓存

JanusGraph包含了事务层（Transaction-Level）、数据库层（Database Level）和后端存储（Storage Backend）缓存，其中第三种位于所配置的后端存储上，下面简单介绍下前2种。

**事务层缓存**包括顶点缓存（Vertex Cache）和索引缓存（Index Cache）。

- **顶点缓存**用于缓存顶点及其相连的边，其大小可通过tx-cache-size 设置，但被修改的vertex被pin在cache中无法替换，所以如果事务修改了很多vertex，那么cache大小会超过设置值。vertex用于加速事务对顶点的多次访问场景。
- **索引缓存**用于缓存使用索引后端进行查询的结果，这样，后续使用相同sql进行查询的时候，就不需要在从索引后端跨网络获取数据，只需从内存中读取即可。最大大小为tx-cache-size 的一半。这个有点像query cache。

**数据库层缓存**在事务间共享缓存数据。对于读多写少的场景，有较好的加速效果。默认关闭，可通过 cache.db-cache=true开启。cache.db-cache-time是影响该性能和查询行为最重要的参数，若仅有一个JanusGraph实例打开存储后端，或仅有该实例有修改存储后端数据，则可以将过期设置为0，表示不过期。

从cache.db-cache-time描述可发现，JanusGraph各个实例之间并没有建立数据同步关系，也就是说，一个实例修改了数据，另一个实例是不感知的。可以把JanusGraph理解为HBase等存储后端的客户端。客户端的缓存是本地缓存，服务器端数据被修改后，客户端的缓存当然不会感知。 cache.db-cache-size 设置数据库层缓存的大小，可以是其所属的JVM内存的百分比，或者是具体的内存byte大小。过大的缓存会引起过多的GC，从而反过来影响性能，默认为50%。cache.db-cache-clean-wait表示本地事务修改了某数据后，等待多少时间将对应数据从缓存中失效掉，从存储后端重新获取。

### JanusGraph数据存储

作为一种分布式图数据库，JanusGraph可将数据切分存储到多台机器上。JanusGraph采用的分片方式是按边切割，而且是对于每一条边，都会被切断。切断后，该边会在起始顶点上和目的顶点上各存储一次。通过这种方式，JanusGraph不管是通过起始顶点，还是从目的顶点，都能快速找到对端顶点。下图所示就是按边切割的方式：

![image](https://user-images.githubusercontent.com/39090338/92709716-bae72100-f389-11ea-9dc2-75bba6c661fb.png)

JanusGraph的数据以**邻接表（adjacency list）**的方式保存：

![image](https://user-images.githubusercontent.com/39090338/92709738-be7aa800-f389-11ea-98ff-b7b373298e52.png)

我们考虑下面最简单的图的数据在HBase中的存储方式：

![image](https://user-images.githubusercontent.com/39090338/92709756-c33f5c00-f389-11ea-91c2-9342d64890cf.png)

下图为按边切分后的结果：

![image](https://user-images.githubusercontent.com/39090338/92709770-c76b7980-f389-11ea-87d5-8b7145e724d9.png)

JanusGraph以点为中心，按切边的方式存储数据。节点的ID作为HBase的Rowkey，节点上的每一个属性和每一条边，作为该Rowkey的一个个独立的Cell。即每一个属性、每一条边，都是一个个独立的KCV结构(Key-Column-Value)。

![image](https://user-images.githubusercontent.com/39090338/92709783-cb979700-f389-11ea-96a2-5f4b98aa5930.png)