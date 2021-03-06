---
logout: post
title: presto架构和内存管理
tags: [presto,sql,all]
---

### presto介绍

Presto 是 Facebook 推出的一个基于Java开发的大数据分布式 SQL 查询引擎，可对从数 G 到数 P 的大数据进行交互式的查询，查询的速度达到商业数据仓库的级别，据称该引擎的性能是 Hive 的 10 倍以上。

presto是一个内存计算引擎，其之所以能在各个内存计算型数据库中脱颖而出，在于以下几点：

> 1. 清晰的架构，是一个能够独立运行的系统，不依赖于任何其他外部系统。例如调度，presto自身提供了对集群的监控，可以根据监控信息完成调度
> 2. 简单的数据结构，列式存储，逻辑行，大部分数据都可以轻易的转化成presto所需要的这种数据结构.
> 3. 丰富的插件接口，完美对接外部存储系统，或者添加自定义的函数。

### presto架构

![https://gitee.com/liurio/image_save/raw/master/presto/presto%E6%9E%B6%E6%9E%84.jpg](https://gitee.com/liurio/image_save/raw/master/presto/presto架构.jpg)

presto采用典型的master-slave结构：

> 1. coordinator: 负责meta管理，worker管理，query的解析和调度。充当master角色
> 2. worker：负责具体计算和读写
> 3. discovery server：通常内嵌在coordinator节点中，也可以单独部署，用于节点心跳。

worker节点启动后会向Discovery Server服务注册，Coordinator从Discovery Server获得可以正常工作的worker节点。如果配置了Hive Connector，需要配置一个Hive MetaStore服务为presto提供hive元信息。

### presto数据模型

![https://gitee.com/liurio/image_save/raw/master/presto/presto%E6%95%B0%E6%8D%AE%E6%A8%A1%E5%9E%8B.jpg](https://gitee.com/liurio/image_save/raw/master/presto/presto数据模型.jpg)

presto采用三层表结构：

> 1. catalog对应某一类数据源，例如hive的数据或mysql
> 2. schema对应mysql中的数据库
> 3. table对应mysql中的表

**presto的存储单元包括：**

- page：多行数据的集合，包含多个列，内部仅提供逻辑行，实际以列式存储。
- block：一列数据，根据不同类型的数据，通常采用不同的编码方式，了解这些编码方式，有助于自己的存储系统对接presto。

**不同类型的block：**

1. array：应用于固定宽度的类型，例如int，long，double。block由两部分组成
   - boolean valueIsNull[]： 示每一行是否有值
   - T values[] 每一行的具体值
2. 可变宽度的string类型的block：由三部分信息组成
   - Slice ： 所有行的数据拼接起来的字符串。
   - int offsets[] :每一行数据的起始便宜位置。每一行的长度等于下一行的起始便宜减去当前行的起始便宜。
   - boolean valueIsNull[] 表示某一行是否有值。如果有某一行无值，那么这一行的便宜量等于上一行的偏移量。
3. 固定宽度的string类型的block：所有行的数据拼接成一长串Slice，每一行的长度固定
4. 字典block：对于某些列，distinct值较少，适合使用字典保存。主要有两部分组成
   - 字典，可以是任意一种类型的block(甚至可以嵌套一个字典block)，block中的每一行按照顺序排序编号。
   - int ids[] 表示每一行数据对应的value在字典中的编号。在查找时，首先找到某一行的id，然后到字典中获取真实的值。

### presto插件管理

presto设计了一个简单的数据存储抽象层，来满足在不同数据存储系统之上都可使用sql进行查询。存储插件（连接器connector）只需奥提供实现以下操作的接口，包括对元数据的获取，获得数据存储的位置，获取数据本身的操作。

1. ConnectorMetadata: 管理表的元数据，表的元数据，partition等信息。在处理请求时，需要获取元信息，以便确认读取的数据的位置。Presto会传入filter条件，以便减少读取的数据的范围。元信息可以从磁盘上读取，也可以缓存在内存中。
2. ConnectorSplit: 一个IO Task处理的数据的集合，是调度的单元。一个split可以对应一个partition，或多个partition。
3. SplitManager : 根据表的meta，构造split。
4. SlsPageSource : 根据split的信息以及要读取的列信息，从磁盘上读取0个或多个page，供计算引擎计算。

### presto内存管理

Presto是一款内存计算型的引擎，所以对于内存管理必须做到精细，才能保证query有序、顺利的执行，部分发生饿死、死锁等情况。

![https://gitee.com/liurio/image_save/raw/master/presto/presto%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.png](https://gitee.com/liurio/image_save/raw/master/presto/presto内存管理.png)

presto把每个worker节点可分配内存（jvm Xmx）分成三份，分别是系统内存池(SystemMemoryPool)，保留内存池(ReservedMemoryPool)和普通内存池(GeneralMemoryPool)。在Presto启动时，它们会随着worker节点初始化时被分配，然后通过服务发现各个worker节点上报给coordinator节点。

![img](https://ask.qcloudimg.com/draft/282243/o8lepm0yem.png)

一个worker节点的内存堆大小可以最大分成两份：系统预留内存+查询内存。而查询内存又分为最大查询内存+其他查询内存。

- **系统预留内存**：worker节点初始化和执行任务必要的内存，包括preto发现服务的定时上报、每个query中task管理数据结构等。使用**resources.reserved-system-memory**配置项配置，默认是worker节点堆大小的0.4。
- **最大查询内存：**coordinator节点会定时调度查看每个query使用的时长和内存，在此过程中会找到耗用内存最大的一个query，并会为此query调度最大的内存使用。这个query可获得各个worker节点最大配置的**专用**最大内存量。使用**query.max-memory-per-node**配置项可以配置，默认是worker节点堆大小的0.1。这个值可根据query监控的peak Mem作为参考设定。并且和保留内存池的大小一致。
- **其他查询内存**：worker节点堆中除了系统预留的内存和最大查询的内存就是其他查询内存。

通过最大查询内存机制，基本解决在使用presto集群时碰到的大部分查询慢和OOM问题。

persto的内存管理总的来说分两部分：

1. query内存管理

   > query划分成很多task， 每个task会有一个线程循环获取task的状态，包括task所用内存。汇总成query所用内存。如果query的汇总内存超过一定大小，则强制终止该query。

2. 机器内存管理

   > coordinator有一个线程，定时的轮训每台机器，查看当前的机器内存状态。

当query内存和机器内存汇总之后，coordinator会挑选出一个内存使用最大的query，分配给Reserved Pool。

### 参考

[https://www.cnblogs.com/tgzhu/p/6033373.html](https://www.cnblogs.com/tgzhu/p/6033373.html)