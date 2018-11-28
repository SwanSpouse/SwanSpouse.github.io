---
title: NSQ 消息消息问题梳理
tags:
    - NSQ
---

#### consumer能否在启动之后，能否消费到之前生产者产生的消息。

1. 首先启动NSQD进程，然后将内存缓冲区设置为0。./nsqd --mem-queue-size=0  
2. 接着创建一个Topic, 并向topic中发送两条数据。

curl -X POST http://127.0.0.1:4151/topic/create\?topic\=lmj
curl -d "111 This is a Message" http://127.0.0.1:4151/pub\?topic\=lmj
curl -d "222 This is a Message" http://127.0.0.1:4151/pub\?topic\=lmj
curl -d "333 This is a Message" http://127.0.0.1:4151/pub\?topic\=lmj

此时在NSQD的目录下 .meta.dat中，看到已经写入3条消息，并且都未进行消费。

3. 然后启动一个NSQ consumer 使用channel 1，来进行消费，看是否能够消费到这几条消息。

启动后，能够正常进行消费。但是消息消费的顺序是:

receive 127.0.0.1:4150 message: 333 This is a Message
receive 127.0.0.1:4150 message: 111 This is a Message
receive 127.0.0.1:4150 message: 222 This is a Message

===???=== 这个有点儿和认知不符啊。为啥呢？磁盘中数据的顺序不应该是相对固定的吗？

4. 再启动一个NSQ consumer 使用channel 2，来进行消费，看是否能够消费到这几条消息。

启动后，无法消费到原有的message。

在发送消息的时候就可以收到新发送的消息。

现在的状态是第一次可以消费到所有的消息。但是后来的consumer就无法消费之前的数据了。

这个时候就需要去翻翻NSQ的文档了。¥
