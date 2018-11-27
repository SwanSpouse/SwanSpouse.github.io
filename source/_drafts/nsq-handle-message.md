---
title: NSQD 消息处理流程
tags:
  - NSQ
---

### NSQD 消息处理整体流程

![NSQD handle message](https://s1.ax1x.com/2018/11/27/FVkOMD.png)

#### memoryMsgChan 和 backendMsgChan

memoryMsgChan：为内存缓冲区，Topic和Channel内存缓冲区的大小可以通过同一变量mem-queue-size来进行配置。

backendMsgChan：为磁盘优先级队列。通过[diskqueue](https://swanspouse.github.io/2018/11/27/nsq-disk-queue/)来实现。发送到backendMsgChan中的消息会通过diskqueue写入到磁盘中。

#### 生产者

作为消息队列的生产者，可以通过HTTP、TCP两种方式向NSQD生产消息。但两种方式相同的是，需要指定生产消息的Topic。

NSQD在接收生产者发送过来的消息的时候，会优先将消息写入Topic的内存缓冲区(memoryMsgChan)中。

如果生产者生产消息的速度过快，或者消费者消费不及时导致memoryMsgChan中的消息积压。当memoryMsgChan已经满了的时候，生产者会直接通过backendMsgChan将数据直接写入磁盘。

NSQD在接收生产者发送来消息的时候，一部分存储在memoryMsgChan、一部分存储在backendMsgChan，其中memoryMsgChan、backendMsgChan中的消息之间保持着原有的先后顺序，但是memoryMsgChan和backendMsgChan之间消息就丢掉了相互之间的先后顺序。所以，NSQD本身是不保证消息的有序性的。但是消费者可以通过消息的ID和时间戳来判别消息之间的先后顺序。

#### 消费者

作为消息队列的消费者。通过和NSQD建立TCP连接，就可以订阅指定Topic的消息进行消费。在订阅消息的时候，需要指定Topic和Channel两个参数。

NSQD在接受到来自消费者的SUB（订阅）请求后，会找到指定Topic下的Channel，若Channel不存在，则会创建一个新的Channel。并通过Channel中的 memoryMsgChan和backendMsgChan向消费者发送消息。

消费者优先消费memoryMsgChan中的消息。

如果消费者消费消息的速度过快，或者生产者生产不及时导致memoryMsgChan中没有数据。消费者会通过backendMsgChan直接消费存储在磁盘上的数据。

#### NSQD

在上述流程中，NSQD负责消息接收、消息发送，以及维护Topic和Channel之间映射关系的功能。其中重要的一环是消息如何从Topic流向Channel。

生产者在生产消息的时候，是以Topic为目标来进行生产的；消费者在消费消息的时候，是以Topic和Channel为目标来进行消费的。所以NSQD中，Topic在接收到生产者消息后，会不断的将消息分发给其下面的各个Channel。

同样也是内存优先的策略。
* 如果是Topic的memoryMsgChan中有消息，Channel的memoryMsgChan中有空位，则直接Topic memoryMsgChan -> Channel memoryMsgChan。
* 其次是 Topic memoryMsgChan -> Channel backendMsgChan 或者 Topic backendMsgChan -> Channel memoryMsgChan
* 最后是 Topic backendMsgChan -> Channel backendMsgChan

即使没有消费者或者生产者的存在，只要Topic和Channel存在，上述过程也是在不断的进行当中的。直到Topic中的消息完全分发给下面所有的Channel。

### NSQD 消息处理相关源码分析


### reference
* NSQ源码 https://github.com/nsqio/nsq
