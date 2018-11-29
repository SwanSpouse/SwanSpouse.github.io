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
* 如果是 Topic 的memoryMsgChan中有消息，Channel的memoryMsgChan中有空位，则直接Topic memoryMsgChan -> Channel memoryMsgChan。
* 其次是 Topic memoryMsgChan -> Channel backendMsgChan 或者 Topic backendMsgChan -> Channel memoryMsgChan
* 最后是 Topic backendMsgChan -> Channel backendMsgChan

即使没有消费者或者生产者的存在，只要Topic和Channel存在，上述过程也是在不断的进行当中的。直到Topic中的消息完全分发给下面所有的Channel。

#### 历史消息问题

在NSQ中，新的消费者是无法重新消费到已经被消费的消息，即使创建了一个新的channel。

因为消费者消费的消息来自Topic的memoryMsgChan和backendMsgChan。而Topic不会记录其下各Channel消费消息的位置。

所以当consumer启动之后，只能消费到新增消息，不能消费到历史消息。

### NSQD 消息处理相关源码分析

这一部分主要说明NSQ消息传递相关的业务逻辑：生产者 -> NSQD -> 消费者。NSQD启动后会初始化HttpServer和TcpServer，来接收客户端请求。这里以TcpServer为例。

#### NSQD接收客户端连接

每当有一个客户端通过TCP的方式连接NSQD，都会通过IOLoop对客户端发送的命令不断进行处理。

```golang
// protocol v2 处理客户端请求
func (p *protocolV2) IOLoop(conn net.Conn) error {
	var err error
	var line []byte
	var zeroTime time.Time
	// 生成ClientID
	clientID := atomic.AddInt64(&p.ctx.nsqd.clientIDSequence, 1)
	// 初始化Client
	client := newClientV2(clientID, conn, p.ctx)
	// 将Client添加到NSQD
	p.ctx.nsqd.AddClient(client.ID, client)
	// synchronize the startup of messagePump in order
	// to guarantee that it gets a chance to initialize
	// goroutine local state derived from client attributes
	// and avoid a potential race with IDENTIFY (where a client
	// could have changed or disabled said attributes)
	// 在这里首先要对client进行一些初始化的工作，为防止这些操作没有做完，所以在这里有个messagePumpStartedChan
	messagePumpStartedChan := make(chan bool)
	// 如果client是生产者，那么会在下面的for循环中不断向 memoryMsgChan 或者 backendMsgChan 写入message。
	// 如果Client是消费者，那么会在messagePump中不断的从memoryMsgChan 或者 backendMsgChan 中读取message。
	go p.messagePump(client, messagePumpStartedChan)
	// 阻塞在这里，直到messagePump这个方法里面的准备工作都已经做好。
	<-messagePumpStartedChan

	// 循环接收客户端发送过来的请求
	for {
		// 设置客户端超时时间
		if client.HeartbeatInterval > 0 {
			client.SetReadDeadline(time.Now().Add(client.HeartbeatInterval * 2))
		} else {
			client.SetReadDeadline(zeroTime)
		}
		// 读取客户端发送的请求
		// ReadSlice does not allocate new space for the data each request
		// ie. the returned slice is only valid until the next call to it
		line, err = client.Reader.ReadSlice('\n')
		if err != nil {
			if err == io.EOF {
				err = nil
			} else {
				err = fmt.Errorf("failed to read command - %s", err)
			}
			// 如果读到EOF，说明客户端已经关闭，则退出循环
			break
		}
		// 忽略末尾的\n trim the '\n'
		line = line[:len(line)-1]
		// 忽略末尾的\r 如果存在的话 optionally trim the '\r'
		if len(line) > 0 && line[len(line)-1] == '\r' {
			line = line[:len(line)-1]
		}
		// 对参数进行解析
		params := bytes.Split(line, separatorBytes)
		p.ctx.nsqd.logf(LOG_DEBUG, "PROTOCOL(V2): [%s] %s", client, params)

		var response []byte
		// 执行客户端发送过来的命令
		response, err = p.Exec(client, params)
		if err != nil {
			ctx := ""
			if parentErr := err.(protocol.ChildErr).Parent(); parentErr != nil {
				ctx = " - " + parentErr.Error()
			}
			p.ctx.nsqd.logf(LOG_ERROR, "[%s] - %s%s", client, err, ctx)

			// 若执行的过程中报错，向客户端返回错误
			sendErr := p.Send(client, frameTypeError, []byte(err.Error()))
			if sendErr != nil {
				p.ctx.nsqd.logf(LOG_ERROR, "[%s] - %s%s", client, sendErr, ctx)
				break
			}
			// errors of type FatalClientErr should forceably close the connection
			if _, ok := err.(*protocol.FatalClientErr); ok {
				break
			}
			continue
		}
		// 如果命令执行正常，则进行回复。
		if response != nil {
			err = p.Send(client, frameTypeResponse, response)
			if err != nil {
				err = fmt.Errorf("failed to send response - %s", err)
				break
			}
		}
	}
	// 读到EOF或者出现错误则退出
	p.ctx.nsqd.logf(LOG_INFO, "PROTOCOL(V2): [%s] exiting ioloop", client)
	conn.Close()
	close(client.ExitChan)
	if client.Channel != nil {
		// 客户端断开后，将client 从 channel中移除
		client.Channel.RemoveClient(client.ID)
	}
	// 将Client从NSQD中移除
	p.ctx.nsqd.RemoveClient(client.ID)
	return err
}
```

#### 客户端向NSQD发送写命令

在上述代码中，NSQD会对客户端发送的命令进行解析并执行。下面来说明，当客户端向NSQD发送PUB/MPUB命令时的一些操作。

```golang
func (p *protocolV2) PUB(client *clientV2, params [][]byte) ([]byte, error) {
    ......
    // 在这里首先会对参数的格式和非法性进行校验。保证参数无误。

    // 获取消息发送的Topic，如果当前Topic不存在，则会创建一个新的Topic
	topic := p.ctx.nsqd.GetTopic(topicName)
    // 创建一个Message实例
	msg := NewMessage(topic.GenerateID(), messageBody)
	// 将Message写入Topic的memoryMsgChan或者backendMsgChan中，这个方法会在下面展开来说
	err = topic.PutMessage(msg)
	if err != nil {
		return nil, protocol.NewFatalClientErr(err, "E_PUB_FAILED", "PUB failed "+err.Error())
	}
    // 会进行一些统计
    ....
    // 消息写入成功，将OK返回给客户端
	return okBytes, nil
}

```

上述代码中很关键的一个环节就是 topic.PutMessage(msg)，下面来说明它的具体实现。

``` golang
/* 通过下面的方法，会将message写入Topic的memoryMsgChan或者backendMsgChan中，
可以看到，这个操作始终是内存优先的策略。如果Topic的memoryMsgChan中没有满，那么会直接写入到memoryMsgChan中；
若Topic的memoryMsgChan中的消息已经满了，才会写入backendMsgChan中。
 */
func (t *Topic) put(m *Message) error {
	select {
	case t.memoryMsgChan <- m:
        // 在这里把message写入到memoryMsgChan当中，也就是写入内存中。
	default:
		// 在这里把消息写到backendMsgChan(diskqueue)中，也就是写入文件中。
		b := bufferPoolGet()
		err := writeMessageToBackend(b, m, t.backend)
		bufferPoolPut(b)
		t.ctx.nsqd.SetHealth(err)
		if err != nil {
			t.ctx.nsqd.logf(LOG_ERROR, "TOPIC(%s) ERROR: failed to write message to backend - %s", t.name, err)
			return err
		}
	}
	return nil
}
```

#### 客户端向NSQD发送读命令

同理，在NSQD收到客户端发来的SUB请求时，


```golang

// 订阅Topic
func (p *protocolV2) SUB(client *clientV2, params [][]byte) ([]byte, error) {
    ....
    // 在这里首先会对参数的格式和非法性进行校验。保证参数无误。

	var channel *Channel
	for {
		// 获取topic和channel，若不存在，则创建相应的Topic和Channel
		topic := p.ctx.nsqd.GetTopic(topicName)
		channel = topic.GetChannel(channelName)
		break
	}

    ....

	// 把channel塞到SubEventChan中，把channel写入到client.SubEventChan中。
	client.SubEventChan <- channel
	// 订阅成功，返回OK
	return okBytes, nil
}

```

上述代码的主要工作，就是在接收到SUB命令时，创建或者获取相应的Topic和Channel，并最终把channel写入到client.SubEventChan中。

```golang

// 将消息不断的发送给Client
func (p *protocolV2) messagePump(client *clientV2, startedChan chan bool) {
    ....

	for {
		// 最初的时候subChannel == nil，在这里将所有变量都进行重置
		if subChannel == nil || !client.IsReadyForMessages() {
			// the client is not ready to receive messages...
			memoryMsgChan = nil
			backendMsgChan = nil
			flusherChan = nil
			flushed = true
		} else if flushed {
			// last iteration we flushed.. do not select on the flusher ticker channel
			// 如果subChannel不为空，并且已经flushed过了，利用channel对memoryMsgChan和backendMsgChan
			// 进行初始化，这时客户端就可以从memoryMsgChan和backendMsgChan中消费消息了。
			memoryMsgChan = subChannel.memoryMsgChan
			backendMsgChan = subChannel.backend.ReadChan()
			flusherChan = nil
		}

		select {
        // 在NSQD接收到来自客户端的SUB命令时，会将客户端订阅的channel写入client.subChannel
		case subChannel = <-subEventChan:
			// you can't SUB anymore
			// 在这里对subChannel赋值，每个Client只能订阅一个Topic，所以这里subEventChan会被置为nil
			subEventChan = nil
		case b := <-backendMsgChan:
			// backendMsgChan 收到消息，然后发送给Client
			msg, err := decodeMessage(b)
			err = p.SendMessage(client, msg)
			if err != nil {
				goto exit
			}
			flushed = false
		case msg := <-memoryMsgChan:
            // memeoryMsgChan 收到消息，然后发送给Client
			err = p.SendMessage(client, msg)
			if err != nil {
				goto exit
			}
			flushed = false
		}
	}
    ....
}
```

#### NSQD中消息从Topic流向Channel

只要存在Topic和Channel并且不处于暂停状态，Topic下面的Message会通过下述方法不断的写入到Channel中。

```golang

// messagePump selects over the in-memory and backend queue and
// writes messages to every channel for this topic
func (t *Topic) messagePump() {

    ....

    // 首先会做一些初始化等工作

	// main message loop
	for {
        // 这里会从memoryMsgChan和backendMsgChan中随机来获取消息，所以NSQD是不保证消息有序的。
		select {
		case msg = <-memoryMsgChan:
		case buf = <-backendChan:
			// 如果从backendChan里面获取的消息，需要首先进行decode
			msg, err = decodeMessage(buf)
			if err != nil {
				t.ctx.nsqd.logf(LOG_ERROR, "failed to decode message - %s", err)
				continue
			}
		case <-t.exitChan:
			goto exit
		}
		// Topic会把自己的消息分发给其下面的所有channel
		// 每个channel都会得到这个消息的复本。所以不同channel之间互不影响。
		for i, channel := range chans {
			chanMsg := msg
			// copy the message because each channel
			// needs a unique instance but...
			// fastpath to avoid copy if its the first channel
			// (the topic already created the first copy)
			if i > 0 {
				chanMsg = NewMessage(msg.ID, msg.Body)
				chanMsg.Timestamp = msg.Timestamp
				chanMsg.deferred = msg.deferred
			}
			// 将消息发送给Topic，和Topic接收消息相似，channel把消息写入其中的memoryMsgChan或者backendMsgChan
			err := channel.PutMessage(chanMsg)
			if err != nil {
				t.ctx.nsqd.logf(LOG_ERROR, "TOPIC(%s) ERROR: failed to put msg(%s) to channel(%s) - %s", t.name, msg.ID, channel.name, err)
			}
		}
	}

    ....
}

```

### reference
* NSQ源码 https://github.com/nsqio/nsq
