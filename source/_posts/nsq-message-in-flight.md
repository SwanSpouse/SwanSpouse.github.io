---
title: NSQ消息Inflight机制
tags:
  - NSQ
date: 2018-12-10 14:49:39
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

首先在Client在连接NSQD成功时，会向NSQD发送SUB命令，订阅想要消费的Topic和channel。

```golang

// 订阅Topic
func (p *protocolV2) SUB(client *clientV2, params [][]byte) ([]byte, error) {

    ...
    // 进行参数校验，并获取topicName和channelName

	var channel *Channel
	for {
		// 获取topic 和 channel
		topic := p.ctx.nsqd.GetTopic(topicName)
		channel = topic.GetChannel(channelName)
		// 将Client 添加到 channel 的Client list中
		channel.AddClient(client.ID, client)
	}
	// 更新Client的状态
	atomic.StoreInt32(&client.State, stateSubscribed)
	client.Channel = channel
	// update message pump
	// 把channel塞到SubEventChan中，这样Client就可以不断消费Topic channel下面的消息了。
	// 一旦protocol中的messagePump接收到这个信号之后，就开始不断向Client发送消息了。
	client.SubEventChan <- channel
	// 订阅成功，返回OK
	return okBytes, nil
}

```

在SubEventChan中收到订阅channel的信息后，messagePump开始不断向Client发送消息。发送之后，在StartInFlightTimeout中将消息写入优先级队列中。

```golang

// 这个函数把Channel下面的消息发送给Client
func (p *protocolV2) messagePump(client *clientV2, startedChan chan bool) {

    ...

	for {
		case subChannel = <-subEventChan:
			// you can't SUB anymore
			// Client发送SUB命令的时候，subEventChan中会收到Client所订阅的channel
			// 在这里对subChannel赋值，每个Client只能订阅一个Topic，所以这里subEventChan会被置为nil
			subEventChan = nil
		case b := <-backendMsgChan:
			// backendMsgChan 收到消息，然后发送给Client
			// 从磁盘中读取出来的消息需要进行解压
			msg, err := decodeMessage(b)
			if err != nil {
				p.ctx.nsqd.logf(LOG_ERROR, "failed to decode message - %s", err)
				continue
			}
			// 将message尝试发送的次数+1
			msg.Attempts++
			// 把message放入到in-flight队列中，等着客户端确认收到消息，再把消息移除
			subChannel.StartInFlightTimeout(msg, client.ID, msgTimeout)
			// 这个里面没有实际发送消息，只是Client对发送数据的统计信息
			client.SendingMessage()
			// 在这里真正向Client发送消息
			err = p.SendMessage(client, msg)
			if err != nil {
				goto exit
			}
			flushed = false
		case msg := <-memoryMsgChan:
			// memoryMsgChan 收到消息，然后发送给Client
			// 将message尝试发送的次数+1
			msg.Attempts++
			// 把message放入到in-flight队列中，等着客户端确认收到消息，再把消息移除
			subChannel.StartInFlightTimeout(msg, client.ID, msgTimeout)
			// 这个里面没有实际发送消息，只是Client对发送数据的统计信息
			client.SendingMessage()
			// 在这里真正向Client发送消息
			err = p.SendMessage(client, msg)
			if err != nil {
				goto exit
			}
			flushed = false
		case <-client.ExitChan:
			goto exit
		}
	}

    ...

```

如果NSQD收到了来自客户端的FIN命令，会把消息从inflight队列中删除。

如果NSQD收到了来自客户端的REQ命令，会把消息重新放入channel的内存或磁盘队列中。

如果NSQD收到了来自客户端的TOUCH命令，会把消息的超时时间进行重置。

如果在超时时间段范围内，没有收到任何来自客户端的消息，NSQD会在queueScanLoop中会启动多个queueScanWorker协程来对消息重新进行处理。

```golang

func (n *NSQD) queueScanWorker(workCh chan *Channel, responseCh chan bool, closeCh chan int) {
	for {
		select {
		case c := <-workCh:
			// 处理相应channel的in-flight queue 和 deferred queue
			now := time.Now().UnixNano()
			dirty := false
			// 处理超时消息
			if c.processInFlightQueue(now) {
				dirty = true
			}
			// 处理延迟消息
			if c.processDeferredQueue(now) {
				dirty = true
			}
			responseCh <- dirty
		// 如果某个协程不幸收到了closeCh的消息，则关闭
		case <-closeCh:
			return
		}
	}
}
```

#### reference

* NSQ源码 https://github.com/nsqio/nsq
