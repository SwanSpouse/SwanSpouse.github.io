---
title: NSQ消息Inflight机制
tags:
    - NSQ
---

### 简介

在NSQ中，使用inflight机制来保证NSQ中消息"at least once"(至少被消费一次)。

在消息发送给Client之后，会将消息以及消息的timeout时间存储到优先级队列中[pqueue](https://swanspouse.github.io/2018/11/29/nsq-pqueue/)。

如果客户端收到该消息，可以使用如下三个命令对此进行回复：
* FIN:   Finish a message，表示成功处理完成。
* REQ:   Re-queue a message，表示消息处理失败，需要重新入队再次进行处理。
* TOUCH: Reset the timeout for an in-flight message，表示需要重置消息的timeout时间。

如果客户端没有收到消息或是收到消息后没有进行任何的回复，则随着到达消息的超时时间，NSQD会将超时的消息重新入队，再次发送给客户端。

NSQD只能保证消息的"at least once"，至于消息的"exactly once"则需要业务端配合来实现。例如通过messageID来判断消息是否被消费过。

#### NSQ消息Inflight机制流程图

![nsq-message-inflight](https://s1.ax1x.com/2018/12/03/FK5p8A.png)

#### 相关源码

![nsqd-queue-scanloop](https://s1.ax1x.com/2018/12/03/FKIEsx.png)


#### reference

* NSQ源码 https://github.com/nsqio/nsq
