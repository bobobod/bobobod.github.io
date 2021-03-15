---
layout: post
title: Elasticsearch读写原理以及性能优化
date: 2020-11-27
tags: elasticsearch
---



### Elasticsearch读写原理以及性能优化

> [参考1](https://juejin.cn/post/6844904007392051207)
>
> [参考2](https://www.javazhiyin.com/22932.html)

### 一 基本概念

---

#### **1.1 索引 index**

Elasticsearch中的“索引”有点像关系数据库中的`数据库`。 它是存储/索引数据的地方。

#### **1.2  分片 shard**

“分片”是Lucene的一个`索引`。 它本身就是一个功能齐全的搜索引擎。

“索引”可以由单个分片组成，但通常由多个分片组成，一部分主分片、一部分副本分片。

ES默认5个主分片，1个副本分片； 副本分片的用途：（1）主节点故障时的故障转移；（2）增加的读取吞吐量。

#### **1.3 分段 segment**

每个分片包含多个“分段”，其中分段是倒排索引。 分段内的doc数量上限是2的31次方。 默认每秒都会生成一个segment文件.

在分片中搜索将依次搜索每个片段，然后将其结果合并到该分片的最终结果中。

查看索引中分段信息的方法：

```
get /test/_segments
```

#### **1.4 translog日志文件**

为了防止elasticsearch宕机造成数据丢失保证可靠存储，es会将每次写入数据同时写到translog日志中。

translog还用于提供`实时CRUD`。 当您尝试按ID检索，更新或删除文档时，它会首先检查translog中是否有任何最近的更改，然后再尝试从相关段中检索文档。 这意味着它始终可以实时访问最新的已知文档版本。

#### 1.5 节点状态

1. green 集群状态正常
2. yellow 主节点正常，副本节点异常
3. red 主节点和副本节点都异常

**集群出现的脑裂问题**

比如我们集群有10个节点，这10个节点可以因为网络原因分为3个节点和7个节点，其中3个节点之间选出一个主节点，7个节点之间也选出一个主节点，当哪天网络恢复后，集群中就会出现两个主节点，这就是**脑裂问题**

### 二 读写原理

---

#### 2.1 Elasticsearch写人数据的过程

1）客户端选择一个node发送请求过去，这个node就是coordinating node（协调节点） 

2）coordinating node，对document进行路由，将请求转发给对应的node（有primary shard） 

3）实际的node上的primary shard处理请求，然后将数据同步到replica node 

4）coordinating node，如果发现primary node和所有replica node都搞定之后，就返回响应结果给客户端

#### 2.2 Elasticsearch读取数据的过程

1）客户端发送请求到任意一个node，成为coordinate node 

2）coordinate node对document进行路由，将请求转发到对应的node，此时会使用round-robin随机轮询算法，在primary shard以及其所有replica中随机选择一个，让读请求负载均衡 

3）接收请求的node返回document给coordinate node 

4）coordinate node返回document给客户端

1.写入document时，每个document会自动分配一个全局唯一的id即doc id，同时也是根据doc id进行hash路由到对应的primary shard上。也可以手动指定doc id，比如用订单id，用户id。

2.读取document时，你可以通过doc id来查询，然后会根据doc id进行hash，判断出来当时把doc id分配到了哪个shard上面去，从那个shard去查询

#### 2.3 Elasticsearch搜索数据过程

es最强大的是做全文检索 

1）客户端发送请求到一个coordinate node 

2）协调节点将搜索请求转发到所有的shard对应的primary shard或replica shard也可以 

3）query phase：每个shard将自己的搜索结果（其实就是一些doc id），返回给协调节点，由协调节点进行数据的合并、排序、分页等操作，产出最终结果 

4）fetch phase：接着由协调节点，根据doc id去各个节点上拉取实际的document数据，最终返回给客户端

搜索的底层原理：倒排索引

#### 2.4 Elasticsearch写数据的底层原理

**流程**

![image](https://user-images.githubusercontent.com/39090338/102595847-3f98c680-4153-11eb-9eb5-d0e5df3b6e63.png)

先写入内存 buffer，在 buffer 里的时候数据是搜索不到的；同时将数据写入 translog 日志文件。

如果 buffer 快满了，或者到一定时间，就会将内存 buffer 数据 `refresh` 到一个新的 `segment file` 中，但是此时数据不是直接进入 `segment file` 磁盘文件，而是先进入 `os cache` 。这个过程就是 `refresh`。

每隔 1 秒钟，es 将 buffer 中的数据写入一个**新的** `segment file`，每秒钟会产生一个**新的磁盘文件** `segment file`，这个 `segment file` 中就存储最近 1 秒内 buffer 中写入的数据。

但是如果 buffer 里面此时没有数据，那当然不会执行 refresh 操作，如果buffer里面有数据，默认 1 秒钟执行一次 refresh 操作，刷入一个新的 segment file 中。

操作系统里面，磁盘文件其实都有一个东西，叫做 `os cache`，即操作系统缓存，就是说数据写入磁盘文件之前，会先进入 `os cache`，先进入操作系统级别的一个内存缓存中去。只要 `buffer` 中的数据被 refresh 操作刷入 `os cache`中，这个数据就可以被搜索到了。

为什么叫 es 是**准实时**的？ 

`NRT`，全称 `near real-time`。默认是每隔 1 秒 refresh 一次的，所以 es 是准实时的，因为写入的数据 1 秒之后才能被看到。可以通过 es 的 `restful api` 或者 `java api`，**手动**执行一次 refresh 操作，就是手动将 buffer 中的数据刷入 `os cache`中，让数据立马就可以被搜索到。只要数据被输入 `os cache` 中，buffer 就会被清空了，因为不需要保留 buffer 了，数据在 translog 里面已经持久化到磁盘去一份了。

重复上面的步骤，新的数据不断进入 buffer 和 translog，不断将 `buffer` 数据写入一个又一个新的 `segment file` 中去，每次 `refresh` 完 buffer 清空，translog保留。随着这个过程推进，translog 会变得越来越大。当 translog 达到一定长度的时候，就会触发 `commit` 操作。

commit 操作发生第一步，就是将 buffer 中现有数据 `refresh` 到 `os cache` 中去，清空 buffer。然后，将一个 `commit point`写入磁盘文件，里面标识着这个 `commit point` 对应的所有 `segment file`，同时强行将 `os cache` 中目前所有的数据都 `fsync` 到磁盘文件中去。最后**清空** 现有 translog 日志文件，重启一个 translog，此时 commit 操作完成。

这个 commit 操作叫做 `flush`。默认 30 分钟自动执行一次 `flush`，但如果 translog 过大，也会触发 `flush`。flush 操作就对应着 commit 的全过程，我们可以通过 es api，手动执行 flush 操作，手动将 os cache 中的数据 fsync 强刷到磁盘上去。

translog 日志文件的作用是什么？

你执行 commit 操作之前，数据要么是停留在 buffer 中，要么是停留在 os cache 中，无论是 buffer 还是 os cache 都是内存，一旦这台机器死了，内存中的数据就全丢了。所以需要将数据对应的操作写入一个专门的日志文件 `translog` 中，一旦此时机器宕机，再次重启的时候，es 会自动读取 translog 日志文件中的数据，恢复到内存 buffer 和 os cache 中去。

translog 其实也是先写入 os cache 的，**默认每隔5秒**刷一次到磁盘中去，所以默认情况下，可能有 5 秒的数据会仅仅停留在 buffer 或者 translog 文件的 os cache 中，如果此时机器挂了，会**丢失** 5 秒钟的数据。但是这样性能比较好，最多丢 5 秒的数据。也可以将 translog 设置成每次写操作必须是直接 `fsync` 到磁盘，但是性能会差很多。

实际上你在这里，如果面试官没有问你 es 丢数据的问题，你可以在这里给面试官炫一把，你说，其实 es 第一是准实时的，数据写入 1 秒后可以搜索到；可能会丢失数据的。有 5 秒的数据，停留在 buffer、translog os cache、segment file os cache 中，而不在磁盘上，此时如果宕机，会导致 5 秒的**数据丢失**。

> 数据写入 segment file 之后，同时就建立好了倒排索引。

#### 2.5 删除/更新数据底层原理

如果是删除操作，commit 的时候会生成一个 `.del` 文件，里面将某个 doc 标识为 `deleted` 状态，那么搜索的时候根据 `.del`文件就知道这个 doc 是否被删除了。

如果是更新操作，就是将原来的 doc 标识为 `deleted` 状态，然后新写入一条数据。

buffer 每次 refresh 一次，就会产生一个 `segment file`，所以默认情况下是 1 秒钟一个 `segment file`，这样下来 `segment file` 会越来越多，此时会定期执行 merge。每次 merge 的时候，会将多个 `segment file` 合并成一个，同时这里会将标识为 `deleted` 的 doc 给**物理删除掉**，然后将新的 `segment file` 写入磁盘，这里会写一个 `commit point`，标识所有新的 `segment file`，然后打开 `segment file` 供搜索使用，同时删除旧的 `segment file`。

### 三 性能调优

#### 3.1 系统层面的调优

系统层面的调优主要是内存的设定与避免交换内存。 ES 安装后默认设置的堆内存是 1GB，这很明显是不够的，那么接下来就会有一个问题出现：我们要设置多少内存给 ES 呢？ 其实这是要看我们集群节点的内存大小，还取决于我们是否在服务器节点上还是否要部署其他服务。 如果内存相对很大，如 64G 及以上，并且不在 ES 集群上部署其他服务，那么建议 ES 内存可以设置为 31G-32G，因为这里有一个 32G 性能瓶颈问题，直白的说就是即使你给了 ES 集群大于 32G 的内存，其性能也不一定会更加优良，甚至会不如设置为 31G-32G 时候的性能。 设置 ES 集群内存的时候，还有一点就是确保堆内存最小值（Xms）与最大值（Xmx）的大小是相同的，防止程序在运行时改变堆内存大小，这是一个很耗系统资源的过程。

禁止swap，一旦允许内存与磁盘的交换，会引起致命的性能问题。 swap空间是一块磁盘空间，操作系统使用这块空间保存从内存中换出的操作系统不常用page数据，这样可以分配出更多的内存做page cache。这样通常会提升系统的吞吐量和IO性能，但同样会产生很多问题。页面频繁换入换出会产生IO读写、操作系统中断，这些都很影响系统的性能。这个值越大操作系统就会更加积极的使用swap空间。 通过： 在elasticsearch.yml 中 bootstrap.memory_lock: true， 以保持JVM锁定内存，保证ES的性能。

#### 3.2 分片与副本

分片 (shard)：ES 是一个分布式的搜索引擎, 索引通常都会分解成不同部分, 分布在不同节点的部分数据就是分片。ES 自动管理和组织分片, 并在必要的时候对分片数据进行再平衡分配, 所以用户基本上不用担心分片的处理细节。创建索引时默认的分片数为 5 个，并且一旦创建不能更改。

副本 (replica)：ES 默认创建一份副本，就是说在 5 个主分片的基础上，每个主分片都相应的有一个副本分片。额外的副本有利有弊，有副本可以有更强的故障恢复能力，但也占了相应副本倍数的磁盘空间。

那我们在创建索引的时候，应该创建多少个分片与副本数呢？

对于副本数，比较好确定，可以根据我们集群节点的多少与我们的存储空间决定，我们的集群服务器多，并且有足够大多存储空间，可以多设置副本数，一般是 1-3 个副本数，如果集群服务器相对较少并且存储空间没有那么宽松，则可以只设定一份副本以保证容灾（副本数可以动态调整）。

对于分片数，是比较难确定的。因为一个索引分片数一旦确定，就不能更改，所以我们在创建索引前，要充分的考虑到，以后我们创建的索引所存储的数据量，否则创建了不合适的分片数，会对我们的性能造成很大的影响。

对于分片数的大小，业界一致认为分片数的多少与内存挂钩，认为 1GB 堆内存对应 20-25 个分片，而一个分片的大小不要超过 50G，这样的配置有助于集群的健康。但是我个人认为这样的配置方法过于死板，我个人在调优 ES 集群的过程中，根据总数据量的大小，设定了相应的分片，保证每一个分片的大小没有超过 50G（大概在 40G 左右），但是相比之前的分片数查询起来，效果并不明显。之后又尝试了增加分片数，发现分片数增多之后，查询速度有了明显的提升，每一个分片的数据量控制在 10G 左右。

查询大量小分片使得每个分片处理数据速度更快了，那是不是分片数越多，我们的查询就越快，ES 性能就越好呢？其实也不是，因为在查询过程中，有一个分片合并的过程，如果分片数不断的增加，合并的时间则会增加，而且随着更多的任务需要按顺序排队和处理，更多的小分片不一定要比查询较小数量的更大的分片更快。如果有多个并发查询，则有很多小碎片也会降低查询吞吐量。

如果现在你的场景是分片数不合适了，但是又不知道如何调整，那么有一个好的解决方法就是按照时间创建索引，然后进行通配查询。如果每天的数据量很大，则可以按天创建索引，如果是一个月积累起来导致数据量很大，则可以一个月创建一个索引。如果要对现有索引进行重新分片，则需要重建索引， 对于每个index的shard数量，可以根据数据总量、写入压力、节点数量等综合考量后设定，然后根据数据增长状态定期检测下shard数量是否合理。

> 腾讯云CES技术团队的推荐方案是： 对于数据量较小（100GB以下）的index，往往写入压力查询压力相对较低，一般设置3~5个shard，number_of_replicas设置为1即可（也就是一主一从，共两副本） 。 对于数据量较大（100GB以上）的index： 一般把单个shard的数据量控制在（20GB~50GB） 让index压力分摊至多个节点：可通过index.routing.allocation.total_shards_per_node参数，强制限定一个节点上该index的shard数量，让shard尽量分配到不同节点上 综合考虑整个index的shard数量，如果shard数量（不包括副本）超过50个，就很可能引发拒绝率上升的问题，此时可考虑把该index拆分为多个独立的index，分摊数据量，同时配合routing使用，降低每个查询需要访问的shard数量。

下面我会介绍一些 ES 关键参数的调优。 有很多场景是，我们的 ES 集群占用了多大的 cpu 使用率，该如何调节呢。cpu 使用率高，有可能是写入导致的，也有可能是查询导致的，那要怎么查看呢？ 

可以先通过 `GET _nodes/{node}/hot_threads` 查看线程栈，查看是哪个线程占用 cpu 高，如果是 `elasticsearch[{node}][search][T#10]` 则是查询导致的，如果是 `elasticsearch[{node}][bulk][T#1]` 则是数据写入导致的。 在实际调优中，cpu 使用率很高，使用固态硬盘（Solid State Disk）替代机械硬盘。SSD 与机械磁盘相比，具有高效的读写速度和稳定性。如果不是 SSD，建议把 `index.merge.scheduler.max_thread_count`: 1 索引 merge 最大线程数设置为 1 个，该参数可以有效调节写入的性能。因为在存储介质上并发写，由于寻址的原因，写入性能不会提升，只会降低。

还有几个重要参数可以进行设置，各位同学可以视自己的集群情况与数据情况而定。

`index.refresh_interval`：这个参数的意思是数据写入后几秒可以被搜索到，默认是 1s。每次索引的 refresh 会产生一个新的 lucene 段, 这会导致频繁的合并行为，如果业务需求对实时性要求没那么高，可以将此参数调大，实际调优告诉我，该参数确实很给力，cpu 使用率直线下降。

`indices.memory.index_buffer_size`：如果我们要进行非常重的高并发写入操作，那么最好将 `indices.memory.index_buffer_size` 调大一些，`index buffer` 的大小是所有的 shard 公用的，对于每个 shard 来说，最多给 512mb，因为再大性能就没什么提升了。ES 会将这个设置作为每个 shard 共享的 index buffer，那些特别活跃的 shard 会更多的使用这个 buffer。默认这个参数的值是 10%，也就是 jvm heap 的 10%。

translog：ES 为了保证数据不丢失，每次 index、bulk、delete、update 完成的时候，一定会触发刷新 translog 到磁盘上。在提高数据安全性的同时当然也降低了一点性能。如果你不在意这点可能性，还是希望性能优先，可以设置如下参数：

```
"index.translog": {
 "sync_interval": "120s",     #sync间隔调高
 "durability": "async",      # 异步更新
 "flush_threshold_size":"1g" #log文件大小
        }
复制代码
```

这样设定的意思是开启异步写入磁盘，并设定写入的时间间隔与大小，有助于写入性能的提升。 replica数目

为了让创建的es index在每台datanode上均匀分布，同一个datanode上同一个index的shard数目不应超过3个。 计算公式: (number_of_shard * (1+number_of_replicas)) < 3*number_of_datanodes 每台机器上分配的shard数目 "index.routing.allocation.total_shards_per_node": "2

#### 3.3 磁盘缓存相关参数

`vm.dirty_background_ratio` 这个参数指定了当文件系统缓存脏页数量达到系统内存百分之多少时（如5%）就会触发`pdflush/flush/kdmflush`等后台回写进程运行，将一定缓存的脏页异步地刷入外存；

```
vm.dirty_ratio
```

该参数则指定了当文件系统缓存脏页数量达到系统内存百分之多少时（如10%），系统不得不开始处理缓存脏页（因为此时脏页数量已经比较多，为了避免数据丢失需要将一定脏页刷入外存）；在此过程中很多应用进程可能会因为系统转而处理文件IO而阻塞。

> 把该参数适当调小，原理通（1）类似。如果cached的脏数据所占比例（这里是占MemTotal的比例）超过这个设置，系统会停止所有的应用层的IO写操作，等待刷完数据后恢复IO。所以万一触发了系统的这个操作，对于用户来说影响非常大的。

```
sysctl -w vm.dirty_ratio=10
sysctl -w vm.dirty_background_ratio=5
复制代码
```

为了将设置永久保存，将上述配置项写入/etc/sysctl.conf文件中

```
vm.dirty_ratio = 10
vm.dirty_background_ratio = 5
复制代码
```

###### merge相关参数

```
"index.merge.policy.floor_segment": "100mb",
 "index.merge.scheduler.max_thread_count": "1",
 "index.merge.policy.min_merge_size": "10mb"
复制代码
```

###### 还有一些超时参数的设置：

```
discovery.zen.ping_timeout 判断 master 选举过程中，发现其他 node 存活的超时设置
discovery.zen.fd.ping_interval 节点被 ping 的频率，检测节点是否存活
discovery.zen.fd.ping_timeout 节点存活响应的时间，默认为 30s，如果网络可能存在隐患，可以适当调大
discovery.zen.fd.ping_retries ping 失败/超时多少导致节点被视为失败，默认为 3
复制代码
```

#### 3.4 Linux系统参数配置

###### 文件句柄

Linux中，每个进程默认打开的最大文件句柄数是1000,对于服务器进程来说，显然太小，通过修改`/etc/security/limits.conf`来增大打开最大句柄数

```
* - nofile 65535
复制代码
```

###### 读优化

①避免大结果集和深翻 在上一篇讲到了集群中的查询流程，例如，要查询从 from 开始的 size 条数据，则需要在每个分片中查询打分排名在前面的 from+size 条数据。 协同节点将收集到的n×(from+size)条数据聚合，再进行一次排序，然后从 from+size 开始返回 size 条数据。 当 from、size 或者 n 中有一个值很大的时候，需要参加排序的数量也会增长，这样的查询会消耗很多 CPU 资源，从而导致效率的降低。 为了提升查询效率，ES 提供了 Scroll 和 Scroll-Scan 这两种查询模式。 Scroll：是为检索大量的结果而设计的。例如，我们需要查询 1～100 页的数据，每页 100 条数据。 如果使用 Search 查询：每次都需要在每个分片上查询得分最高的 from+100 条数据，然后协同节点把收集到的 n×(from+100)条数据聚合起来再进行一次排序。 每次返回 from+1 开始的 100 条数据，并且要重复执行 100 次。 如果使用 Scroll 查询：在各个分片上查询 10000 条数据，协同节点聚合 n×10000 条数据进行合并、排序，并将排名前 10000 的结果快照起来。这样做的好处是减少了查询和排序的次数。

###### 其他建议

插入索引自动生成 id：当写入端使用特定的 id 将数据写入 ES 时，ES 会检查对应的索引下是否存在相同的 id，这个操作会随着文档数量的增加使消耗越来越大，所以如果业务上没有硬性需求建议使用 ES 自动生成的 id，加快写入速率。

避免稀疏索引：索引稀疏之后，会导致索引文件增大。ES 的 keyword，数组类型采用 doc_values 结构，即使字段是空值，每个文档也会占用一定的空间，所以稀疏索引会造成磁盘增大，导致查询和写入效率降低。

###### 参数调优

```
index.merge.scheduler.max_thread_count:1 # 索引 merge 最大线程数
indices.memory.index_buffer_size:30% # 内存
index.translog.durability:async # 这个可以异步写硬盘，增大写的速度
index.translog.sync_interval:120s #translog 间隔时间
discovery.zen.ping_timeout:120s # 心跳超时时间
discovery.zen.fd.ping_interval:120s     # 节点检测时间
discovery.zen.fd.ping_timeout:120s     #ping 超时时间
discovery.zen.fd.ping_retries:6 # 心跳重试次数
thread_pool.bulk.size:20 # 写入线程个数 由于我们查询线程都是在代码里设定好的，我这里只调节了写入的线程数
thread_pool.bulk.queue_size:1000 # 写入线程队列大小
index.refresh_interval:300s #index 刷新间隔
bootstrap.memory_lock: true#以保持JVM锁定内存，保证ES的性能。 
复制代码
```

#### 3.5 关于重建索引

在重建索引之前，首先要考虑一下重建索引的必要性，因为重建索引是非常耗时的。 ES 的 reindex api 不会去尝试设置目标索引，不会复制源索引的设置，所以我们应该在运行_reindex 操作之前设置目标索引，包括设置映射（mapping），分片，副本等。

第一步，和创建普通索引一样创建新索引。当数据量很大的时候，需要设置刷新时间间隔，把 `refresh_intervals` 设置为-1，即不刷新,number_of_replicas 副本数设置为 0（因为副本数可以动态调整，这样有助于提升速度）。

```json
{ 
"settings": {
 "number_of_shards": "50",
 "number_of_replicas": "0", 
 "index": { "refresh_interval": "-1" }
              } 

"mappings":
 {
    
  }
}
```

第二步，调用 reindex 接口，建议加上 `wait_for_completion=false` 的参数条件，这样 reindex 将直接返回 taskId。

```json
POST _reindex?wait_for_completion=false { "source": { "index": "old_index",   //原有索引
  "size": 5000            //一个批次处理的数据量
}, "dest": { "index": "new_index",   //目标索引
}
}
```

第三步，等待。可以通过 `GET _tasks?detailed=true&actions=*reindex` 来查询重建的进度。如果要取消 task 则调用`_tasks/node_id:task_id/_cancel`。

第四步，删除旧索引，释放磁盘空间。重建索引的时候，在参数里加上上一次重建索引的时间戳，直白的说就是，比如我们的数据是 100G，这时候我们重建索引了，但是这个 100G 在增加，那么我们重建索引的时候，需要记录好重建索引的时间戳，记录时间戳的目的是下一次重建索引跑任务的时候不用全部重建，只需要在此时间戳之后的重建就可以，如此迭代，直到新老索引数据量基本一致，把数据流向切换到新索引的名字。

```
POST /_reindex
{ 
"conflicts": "proceed",          //意思是冲突以旧索引为准，直接跳过冲突，否则会抛出异常，停止task
    "source": { "index": "old_index"         //旧索引
        "query": { "constant_score" : 
                      { "filter" : { 
                          "range" : { "data_update_time" : 
                                          { "gte" : 123456789   //reindex开始时刻前的毫秒时间戳
                                              }
                        }
                    }
                }
            }
        }, 
"dest": { "index": "new_index",       //新索引
        "version_type": "external"  //以旧索引的数据为准
 }
}
```