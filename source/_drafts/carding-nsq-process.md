---
title: NSQ 消息消息问题梳理
tags:
    - NSQ
---


#### 解决topic和channel之间传递数据的问题

启动了NSQD, 然后把mem-queue-size 设置成了3。然后往里面写了5条数据，查看.dat文件发现:

2
0,0
0,102

这个是符合我的正常认知的。3条消息在内存中，然后2条消息已经被持久化到磁盘中。

[nsqd] 2018/11/28 22:30:14.331227 INFO: nsqd v1.1.0 (built w/go1.8)
[nsqd] 2018/11/28 22:30:14.331329 INFO: ID: 294
[nsqd] 2018/11/28 22:30:14.331359 INFO: NSQ: persisting topic/channel metadata to nsqd.dat
[nsqd] 2018/11/28 22:30:14.332137 INFO: TCP: listening on [::]:4150
[nsqd] 2018/11/28 22:30:14.332189 INFO: HTTP: listening on [::]:4151
[nsqd] 2018/11/28 22:30:24.923213 INFO: TOPIC(lmj): created
[nsqd] 2018/11/28 22:30:24.923288 INFO: MINGJI DEBUG ===========> topic:current message was sent to memoryMsgChan. 111 This is a Message
[nsqd] 2018/11/28 22:30:24.923395 INFO: NSQ: persisting topic/channel metadata to nsqd.dat
[nsqd] 2018/11/28 22:30:24.995957 INFO: MINGJI DEBUG ===========> topic:current message was sent to memoryMsgChan. 222 This is a Message
[nsqd] 2018/11/28 22:30:25.986699 INFO: MINGJI DEBUG ===========> topic:current message was sent to memoryMsgChan. 333 This is a Message
[nsqd] 2018/11/28 22:31:04.599047 INFO: MINGJI DEBUG ===========> topic:current message was sent to backendMsgChan. 444 This is a Message
[nsqd] 2018/11/28 22:31:04.599212 INFO: DISKQUEUE(lmj): writeOne() opened lmj.diskqueue.000000.dat
[nsqd] 2018/11/28 22:31:04.599266 INFO: DISKQUEUE(lmj): readOne() opened lmj.diskqueue.000000.dat
[nsqd] 2018/11/28 22:31:34.349515 INFO: MINGJI DEBUG ===========> topic:current message was sent to backendMsgChan. 555 This is a Message

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

这个时候就需要去翻翻NSQ的文档了。 这个和下面是一个问题。

#### NSQD启动之后，除了第一个consumer之外无法消费到memoryMsgChan中的消息

启动NSQD，设置mem-queue-size=3，然后往里面写5条数据。

启动consumer，channel-1 能够消费到5条数据

启动consumer, channel-2 一条数据都消费不到。为啥呢？不是应该能消费到磁盘里面的两条数据吗？

就是应该一条数据都消费不到。因为消费到哪里的这个标识是存储在Topic里面的。





















#### end
