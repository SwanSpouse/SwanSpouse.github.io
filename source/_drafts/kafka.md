---
title: kafka
tags:
    - MQ
    - Kafka
---

### kafka概述

Kafka 起初是由 LinkedIn 公司开发的一个分布式的消息系统，后成为 Apache 的一部分，它使用 Scala 编写，以可水平扩展和高吞吐率而被广泛使用。目前越来越多的开源分布式处理系统如 Cloudera、Apache Storm、Spark 等都支持与 Kafka 集成。

Kafka 与传统消息系统相比，有以下不同：
* 它被设计为一个分布式系统，易于向外扩展；
* 它同时为发布和订阅提供高吞吐量；
* 它支持多订阅者，当失败时能自动平衡消费者；
* 它将消息持久化到磁盘，因此可用于批量消费，例如 ETL 以及实时应用程序。

### kafka体系架构

![kafka架构](https://s2.ax1x.com/2019/01/23/kEYUyt.jpg)

如上图所示，一个典型的 Kafka 体系架构包括若干 Producer（可以是服务器日志，业务数据，页面前端产生的 page view 等等），若干 broker（Kafka 支持水平扩展，一般 broker 数量越多，集群吞吐率越高），若干 Consumer (Group)，以及一个 Zookeeper 集群。Kafka 通过 Zookeeper 管理集群配置，选举 leader，以及在 consumer group 发生变化时进行 rebalance。Producer 使用 push(推) 模式将消息发布到 broker，Consumer 使用 pull(拉) 模式从 broker 订阅并消费消息。

#### 主要角色

Broker：一个Kafka节点就是一个Broker，一个或者多个Broker组成Kafka集群。
Topic：Kafka根据Topic对消息进行归类，发布到Kafka集群的每条消息都需要指定一个Topic
Producer：消息生产者，向Broker发送消息的客户端。
Consumer：消息消费者，从Broker读取消息的客户端。
ConsumerGroup： 每个Consumer都属于一个特定的ConsumerGroup，一条消息可以发送到多个不同的ConsumerGroup，但是一个ConsumerGroup中只能有一个Consumer能够消费该消息。
Partition：物理上的概念，一个Topic可以为多个Partition，每个Partition内部有序。

#### Topic和Partition

一个 topic 可以认为一个一类消息，每个 topic 将被分成多个 partition，每个 partition 在存储层面是 append log 文件。任何发布到此 partition 的消息都会被追加到 log 文件的尾部，每条消息在文件中的位置称为 offset(偏移量)，offset 为一个 long 型的数字，它唯一标记一条消息。每条消息都被 append 到 partition 中，是顺序写磁盘，因此效率非常高（经验证，顺序写磁盘效率比随机写内存还要高，这是 Kafka 高吞吐率的一个很重要的保证）。

![kafka写message](https://s2.ax1x.com/2019/01/23/kEthjI.png)

每一条消息被发送到 broker 中，会根据 partition 规则选择被存储到哪一个 partition。如果 partition 规则设置的合理，所有消息可以均匀分布到不同的 partition 里，这样就实现了水平扩展。（如果一个 topic 对应一个文件，那这个文件所在的机器 I/O 将会成为这个 topic 的性能瓶颈，而 partition 解决了这个问题）。在创建 topic 时可以在 $KAFKA_HOME/config/server.properties 中指定这个 partition 的数量，当然可以在 topic 创建之后去修改 partition 的数量。

在发送一条消息时，可以指定这个消息的 key，producer 根据这个 key 和 partition 机制来判断这个消息发送到哪个 partition。partition 机制可以通过指定 producer 的 partition.class 这一参数来指定，该 class 必须实现 kafka.producer.Partitioner 接口。

### Kafka高可用性

Kafka 的高可靠性的保障来源于其健壮的副本（replication）策略。通过调节其副本相关参数，可以使得 Kafka 在性能和可靠性之间运转的游刃有余。Kafka 从 0.8.x 版本开始提供 partition 级别的复制,replication 的数量可以在 $KAFKA_HOME/config/server.properties 中配置（default.replication.refactor）。

#### Kafka文件存储机制

Kafka 中消息是以 topic 进行分类的，生产者通过 topic 向 Kafka broker 发送消息，消费者通过 topic 读取数据。然而 topic 在物理层面又能以 partition 为分组，一个 topic 可以分成若干个 partition。partition 还可以细分为 segment，一个 partition 物理上由多个 segment 组成。

```xml
drwxr-xr-x 2 root root 4096 Apr 10 16:10 topic_vms_test-0
drwxr-xr-x 2 root root 4096 Apr 10 16:10 topic_vms_test-1
drwxr-xr-x 2 root root 4096 Apr 10 16:10 topic_vms_test-2
drwxr-xr-x 2 root root 4096 Apr 10 16:10 topic_vms_test-3
```

在 Kafka 文件存储中，同一个 topic 下有多个不同的 partition，每个 partition 为一个目录，partition 的名称规则为：topic 名称 + 有序序号，第一个序号从 0 开始计，最大的序号为 partition 数量减 1，partition 是实际物理上的概念，而 topic 是逻辑上的概念。

当 Kafka producer 不断发送消息，必然会引起 partition 文件的无限扩张，这样对于消息文件的维护以及已经被消费的消息的清理带来严重的影响，所以这里以 segment 为单位又将 partition 细分。每个 partition(目录) 相当于一个巨型文件被平均分配到多个大小相等的 segment(段) 数据文件中（每个 segment 文件中消息数量不一定相等）这种特性也方便 old segment 的删除，即方便已被消费的消息的清理，提高磁盘的利用率。每个 partition 只需要支持顺序读写就行，segment 的文件生命周期由服务端配置参数（log.segment.bytes，log.roll.{ms,hours}等若干参数）决定。

segment 文件由两部分组成，分别为“.index”文件和“.log”文件，分别表示为 segment 索引文件和数据文件。这两个文件的命令规则为：partition 全局的第一个 segment 从 0 开始，后续每个 segment 文件名为上一个 segment 文件最后一条消息的 offset 值，数值大小为 64 位，20 位数字字符长度，没有数字用 0 填充，如下：

```xml
00000000000000000000.index
00000000000000000000.log
00000000000000170410.index
00000000000000170410.log
00000000000000239430.index
00000000000000239430.log
```

以上面的 segment 文件为例，展示出 segment：00000000000000170410 的“.index”文件和“.log”文件的对应的关系，如下图：

![kafka segement](https://s2.ax1x.com/2019/01/23/kEaQcd.png)

如上图，“.index”索引文件存储大量的元数据，“.log”数据文件存储大量的消息，索引文件中的元数据指向对应数据文件中 message 的物理偏移地址。其中以“.index”索引文件中的元数据 [3, 348] 为例，在“.log”数据文件表示第 3 个消息，即在全局 partition 中表示 170410+3=170413 个消息，该消息的物理偏移地址为 348。

那么如何从 partition 中通过 offset 查找 message 呢？以上图为例，读取 offset=170418 的消息，首先查找 segment 文件，其中 00000000000000000000.index 为最开始的文件，第二个文件为 00000000000000170410.index（起始偏移为 170410+1=170411），而第三个文件为 00000000000000239430.index（起始偏移为 239430+1=239431），所以这个 offset=170418 就落到了第二个文件之中。其他后续文件可以依次类推，以其实偏移量命名并排列这些文件，然后根据二分查找法就可以快速定位到具体文件位置。其次根据 00000000000000170410.index 文件中的 [8,1325] 定位到 00000000000000170410.log 文件中的 1325 的位置进行读取。


### reference
* https://www.infoq.cn/article/depth-interpretation-of-kafka-data-reliability
