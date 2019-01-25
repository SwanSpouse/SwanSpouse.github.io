---
title: Kafka简介
tags:
  - MQ
  - Kafka
date: 2019-01-23 22:40:15
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

#### ZeroCopy

当数据源是一个Channel,数据接收端也是一个Channel(SocketChannel),则通过ZeroCopy方式进行数据传输，是直接在内核态进行的，避免拷贝数据导致的内核态和用户态的多次切换。

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

#### 复制原理和同步方式

Kafka 中 topic 的每个 partition 有一个预写式的日志文件，虽然 partition 可以继续细分为若干个 segment 文件，但是对于上层应用来说可以将 partition 看成最小的存储单元（一个有多个 segment 文件拼接的“巨型”文件），每个 partition 都由一些列有序的、不可变的消息组成，这些消息被连续的追加到 partition 中。

![kafka segement](https://s2.ax1x.com/2019/01/23/kV2iQ0.png)

LEO: LogEndOffset, 表示每个 partition 的 log 最后一条 Message 的位置。
HW: HighWatermark, 是指 consumer 能够看到的此 partition 的位置。

为了提高消息的可靠性，Kafka 每个 topic 的 partition 有 N 个副本（replicas），其中 N(大于等于 1) 是 topic 的复制因子（replica fator）的个数。Kafka 通过多副本机制实现故障自动转移，当 Kafka 集群中一个 broker 失效情况下仍然保证服务可用。在 Kafka 中发生复制时确保 partition 的日志能有序地写到其他节点上，N 个 replicas 中，其中一个 replica 为 leader，其他都为 follower, leader 处理 partition 的所有读写请求，与此同时，follower 会被动定期地去复制 leader 上的数据。

如下图所示，Kafka 集群中有 4 个 broker, 某 topic 有 3 个 partition, 且复制因子即副本个数也为 3：

![kafka partition](https://s2.ax1x.com/2019/01/23/kV2uWR.jpg)

Kafka 提供了数据复制算法保证，如果 leader 发生故障或挂掉，一个新 leader 被选举并被接受客户端的消息成功写入。Kafka 确保从同步副本列表中选举一个副本为 leader，或者说 follower 追赶 leader 数据。leader 负责维护和跟踪 ISR(In-Sync Replicas 的缩写，表示副本同步队列) 中所有 follower 滞后的状态。当 producer 发送一条消息到 broker 后，leader 写入消息并复制到所有 follower。消息提交之后才被成功复制到所有的同步副本。消息复制延迟受最慢的 follower 限制，重要的是快速检测慢副本，如果 follower“落后”太多或者失效，leader 将会把它从 ISR 中删除。


#### ISR

ISR (In-Sync Replicas)，这个是指副本同步队列。副本数对 Kafka 的吞吐率是有一定的影响，但极大的增强了可用性。默认情况下 Kafka 的 replica 数量为 1，即每个 partition 都有一个唯一的 leader，为了确保消息的可靠性，通常应用中将其值 (由 broker 的参数 offsets.topic.replication.factor 指定) 大小设置为大于 1，比如 3。 所有的副本（replicas）统称为 Assigned Replicas，即 AR。

ISR 是 AR 中的一个子集，由 leader 维护 ISR 列表，follower 从 leader 同步数据有一些延迟（包括延迟时间 replica.lag.time.max.ms 和延迟条数 replica.lag.max.messages 两个维度, 当前最新的版本 0.10.x 中只支持 replica.lag.time.max.ms 这个维度），任意一个超过阈值都会把 follower 剔除出 ISR, 存入 OSR（Outof-Sync Replicas）列表，新加入的 follower 也会先存放在 OSR 中。AR=ISR+OSR。

Kafka 0.10.x 版本后移除了 replica.lag.max.messages 参数，只保留了 replica.lag.time.max.ms 作为 ISR 中副本管理的参数。为什么这样做呢？replica.lag.max.messages 表示当前某个副本落后 leaeder 的消息数量超过了这个参数的值，那么 leader 就会把 follower 从 ISR 中删除。假设设置 replica.lag.max.messages=4，那么如果 producer 一次传送至 broker 的消息数量都小于 4 条时，因为在 leader 接受到 producer 发送的消息之后而 follower 副本开始拉取这些消息之前，follower 落后 leader 的消息数不会超过 4 条消息，故此没有 follower 移出 ISR，所以这时候 replica.lag.max.message 的设置似乎是合理的。

但是 producer 发起瞬时高峰流量，producer 一次发送的消息超过 4 条时，也就是超过 replica.lag.max.messages，此时 follower 都会被认为是与 leader 副本不同步了，从而被踢出了 ISR。但实际上这些 follower 都是存活状态的且没有性能问题。那么在之后追上 leader, 并被重新加入了 ISR。于是就会出现它们不断地剔出 ISR 然后重新回归 ISR，这无疑增加了无谓的性能损耗。而且这个参数是 broker 全局的。设置太大了，影响真正“落后”follower 的移除；设置的太小了，导致 follower 的频繁进出。无法给定一个合适的 replica.lag.max.messages 的值，故此，新版本的 Kafka 移除了这个参数。

上面还涉及到一个概念，即 HW。HW 俗称高水位，HighWatermark 的缩写，取一个 partition 对应的 ISR 中最小的 LEO 作为 HW，consumer 最多只能消费到 HW 所在的位置。另外每个 replica 都有 HW,leader 和 follower 各自负责更新自己的 HW 的状态。对于 leader 新写入的消息，consumer 不能立刻消费，leader 会等待该消息被所有 ISR 中的 replicas 同步后更新 HW，此时消息才能被 consumer 消费。这样就保证了如果 leader 所在的 broker 失效，该消息仍然可以从新选举的 leader 中获取。对于来自内部 broKer 的读取请求，没有 HW 的限制。

Kafka 的 ISR 的管理最终都会反馈到 Zookeeper 节点上。具体位置为：/brokers/topics/[topic]/partitions/[partition]/state。目前有两个地方会对这个 Zookeeper 的节点进行维护：

* Controller 来维护：Kafka 集群中的其中一个 Broker 会被选举为 Controller，主要负责 Partition 管理和副本状态管理，也会执行类似于重分配 partition 之类的管理任务。在符合某些特定条件下，Controller 下的 LeaderSelector 会选举新的 leader，ISR 和新的 leader_epoch 及 controller_epoch 写入 Zookeeper 的相关节点中。同时发起 LeaderAndIsrRequest 通知所有的 replicas。

* leader 来维护：leader 有单独的线程定期检测 ISR 中 follower 是否脱离 ISR, 如果发现 ISR 变化，则会将新的 ISR 的信息返回到 Zookeeper 的相关节点中。


#### 数据可靠性和持久性保证

当 producer 向 leader 发送数据时，可以通过 request.required.acks 参数来设置数据可靠性的级别：

* 1（默认）：这意味着 producer 在 ISR 中的 leader 已成功收到的数据并得到确认后发送下一条 message。如果 leader 宕机了，则会丢失数据。
* 0：这意味着 producer 无需等待来自 broker 的确认而继续发送下一批消息。这种情况下数据传输效率最高，但是数据可靠性确是最低的。
* -1：producer 需要等待 ISR 中的所有 follower 都确认接收到数据后才算一次发送完成，可靠性最高。但是这样也不能保证数据不丢失，比如当 ISR 中只有 leader 时（前面 ISR 那一节讲到，ISR 中的成员由于某些情况会增加也会减少，最少就只剩一个 leader），这样就变成了 acks=1 的情况。

如果要提高数据的可靠性，在设置 request.required.acks=-1 的同时，也要 min.insync.replicas 这个参数 (可以在 broker 或者 topic 层面进行设置) 的配合，这样才能发挥最大的功效。min.insync.replicas 这个参数设定 ISR 中的最小副本数是多少，默认值为 1，当且仅当 request.required.acks 参数设置为 -1 时，此参数才生效。如果 ISR 中的副本数少于 min.insync.replicas 配置的数量时，客户端会返回异常：org.apache.kafka.common.errors.NotEnoughReplicasExceptoin: Messages are rejected since there are fewer in-sync replicas than required。

#### Leader选举策略

一条消息只有被 ISR 中的所有 follower 都从 leader 复制过去才会被认为已提交。这样就避免了部分数据被写进了 leader，还没来得及被任何 follower 复制就宕机了，而造成数据丢失。而对于 producer 而言，它可以选择是否等待消息 commit，这可以通过 request.required.acks 来设置。这种机制确保了只要 ISR 中有一个或者以上的 follower，一条被 commit 的消息就不会丢失。

有一个很重要的问题是当 leader 宕机了，怎样在 follower 中选举出新的 leader，因为 follower 可能落后很多或者直接 crash 了，所以必须确保选择“最新”的 follower 作为新的 leader。一个基本的原则就是，如果 leader 不在了，新的 leader 必须拥有原来的 leader commit 的所有消息。这就需要做一个折中，如果 leader 在表名一个消息被 commit 前等待更多的 follower 确认，那么在它挂掉之后就有更多的 follower 可以成为新的 leader，但这也会造成吞吐率的下降。

一种非常常用的选举 leader 的方式是“少数服从多数”，Kafka 并不是采用这种方式。这种模式下，如果我们有 2f+1 个副本，那么在 commit 之前必须保证有 f+1 个 replica 复制完消息，同时为了保证能正确选举出新的 leader，失败的副本数不能超过 f 个。这种方式有个很大的优势，系统的延迟取决于最快的几台机器，也就是说比如副本数为 3，那么延迟就取决于最快的那个 follower 而不是最慢的那个。

“少数服从多数”的方式也有一些劣势，为了保证 leader 选举的正常进行，它所能容忍的失败的 follower 数比较少，如果要容忍 1 个 follower 挂掉，那么至少要 3 个以上的副本，如果要容忍 2 个 follower 挂掉，必须要有 5 个以上的副本。也就是说，在生产环境下为了保证较高的容错率，必须要有大量的副本，而大量的副本又会在大数据量下导致性能的急剧下降。这种算法更多用在 Zookeeper 这种共享集群配置的系统中而很少在需要大量数据的系统中使用的原因。HDFS 的 HA 功能也是基于“少数服从多数”的方式，但是其数据存储并不是采用这样的方式。

实际上，leader 选举的算法非常多，比如 Zookeeper 的Zab、Raft以及Viewstamped Replication。而 Kafka 所使用的 leader 选举算法更像是微软的PacificA算法。

Kafka 在 Zookeeper 中为每一个 partition 动态的维护了一个 ISR，这个 ISR 里的所有 replica 都跟上了 leader，只有 ISR 里的成员才能有被选为 leader 的可能（unclean.leader.election.enable=false）。在这种模式下，对于 f+1 个副本，一个 Kafka topic 能在保证不丢失已经 commit 消息的前提下容忍 f 个副本的失败，在大多数使用场景下，这种模式是十分有利的。事实上，为了容忍 f 个副本的失败，“少数服从多数”的方式和 ISR 在 commit 前需要等待的副本的数量是一样的，但是 ISR 需要的总的副本的个数几乎是“少数服从多数”的方式的一半。

上文提到，在 ISR 中至少有一个 follower 时，Kafka 可以确保已经 commit 的数据不丢失，但如果某一个 partition 的所有 replica 都挂了，就无法保证数据不丢失了。这种情况下有两种可行的方案：

* 等待 ISR 中任意一个 replica“活”过来，并且选它作为 leader
* 选择第一个“活”过来的 replica（并不一定是在 ISR 中）作为 leader

这就需要在可用性和一致性当中作出一个简单的抉择。如果一定要等待 ISR 中的 replica“活”过来，那不可用的时间就可能会相对较长。而且如果 ISR 中所有的 replica 都无法“活”过来了，或者数据丢失了，这个 partition 将永远不可用。选择第一个“活”过来的 replica 作为 leader, 而这个 replica 不是 ISR 中的 replica, 那即使它并不保障已经包含了所有已 commit 的消息，它也会成为 leader 而作为 consumer 的数据源。默认情况下，Kafka 采用第二种策略，即 unclean.leader.election.enable=true，也可以将此参数设置为 false 来启用第一种策略。


#### Leader选举过程

最简单最直观的方案是，leader在zk上创建一个临时节点，所有Follower对此节点注册监听，当leader宕机时，此时ISR里的所有Follower都尝试创建该节点，而创建成功者（Zookeeper保证只有一个能创建成功）即是新的Leader，其它Replica即为Follower。

实际上的实现思路也是这样，只是优化了下，多了个代理控制管理类（controller）。引入的原因是，当kafka集群业务很多，partition达到成千上万时，当broker宕机时，造成集群内大量的调整，会造成大量Watch事件被触发，Zookeeper负载会过重。zk是不适合大量写操作的。


### reference
* https://www.infoq.cn/article/depth-interpretation-of-kafka-data-reliability
