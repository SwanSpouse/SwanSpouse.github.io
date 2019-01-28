---
title: LSM Tree
tags:
  - LSM Tree
  - distributed system
date: 2019-01-28 12:25:53
---


## LSM Tree (Log-Structured Merge Tree）

LSM Tree是为了优化数据库写性能而出现的，相较于传统的B+树，它减少了磁盘随机读取的需求，从而在一定程度上改善了数据库的写能力，当然在一定程度上牺牲了数据库的读能力。

与其说LSM Tree是一种树，不如说它是通过传统索引组织有序文件或内存块的一种方式。

LSM Tree的节点可以分为两种：

* MemTable: 保存在内存中的小树，称之为MemTable，一般可以是红黑树、跳跃表，甚至可以是B树。
* SSTable: 保存在磁盘上的大树，称之为SSTable，通常是B树或起变种。

### LSM Tree和 B+树对比

传统的B+树的缺陷就是在访问节点时涉及到了大量的磁盘随机读写，因为你无法保证节点常驻内存，尤其是当B+树管理的索引量很大的时候。这导致数据库读写性能急剧下降。

LSM树采取的做法就是通过引入多部件索引来减少磁盘随机读写的需求，如何减少呢？在大量插入情况下我们周期性地选取两部分索引进行合并，并且把合并后的有序文件（或内存块）添加到磁盘尾部（或成为新文件），修改节点信息以保证索引树的正确和完整，并且周期性地回收失效索引。

### LSM Tree 基本原理

![LSM树原理](https://s2.ax1x.com/2019/01/28/kKIsVH.jpg)

写操作直接作用于MemTable, 因此写入性能接近写内存。

每层SSTable文件到达一定条件后，进行合并操作，然后放置到更高层。合并操作在实现上一般是策略驱动、可插件化的。比如Cassandra的合并策略可以选择SizeTieredCompactionStrategy或LeveledCompactionStrategy.

#### LSM Tree 合并

![LSM Tree Compact](https://s2.ax1x.com/2019/01/28/kKIRRP.jpg)

SSTable合并类似于简单的归并排序：根据key值确定要merge的文件，然后进行合并。因此，合并一个文件到更高层，可能会需要写多个文件。存在一定程度的写放大。是非常昂贵的I/O操作行为。Cassandra除了提供策略进行合并文件的选择，还提供了合并时I/O的限制，以期减少合并操作对上层业务的影响。

#### Bloom Filter

![LSM Bloom Filter](https://s2.ax1x.com/2019/01/28/kKIoZQ.jpg)

读操作优先判断key是否在MemTable, 如果不在的话，则把覆盖该key range的所有SSTable都查找一遍。简单，但是低效。因此，在工程实现上，一般会为SSTable加入索引。可以是一个key-offset索引（类似于kafka的index文件），也可以是布隆过滤器（Bloom Filter）。布隆过滤器有一个特性：如果bloom说一个key不存在，就一定不存在，而当bloom说一个key存在于这个文件，可能是不存在的。实现层面上，布隆过滤器就是key--比特位的映射。理想情况下，当然是一个key对应一个比特实现全映射，但是太消耗内存。因此，一般通过控制假阳性概率来节约内存，代价是牺牲了一定的读性能。对于我们的应用场景，我们将该概率从0.99降低到0.8，布隆过滤器的内存消耗从2GB+下降到了300MB，数据读取速度有所降低，但在感知层面可忽略。

### reference

* https://liudanking.com/arch/lsm-tree-summary/
* https://zhuanlan.zhihu.com/p/26168026
