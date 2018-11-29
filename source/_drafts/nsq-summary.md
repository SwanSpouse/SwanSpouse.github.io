---
title: nsq 源码解读提纲
tags:
    - NSQ
---

nsq 代码目录结构。
nsqd 相关解读。 topic channel 等等这些基本概念。
这里先说diskqueue吧。

* 关于disk queue 的思考，disk queue中wirteChan和readChan都是不带缓冲区的channel。为啥呢？带有缓冲区不是可以提高消息处理的效率吗？是处于消息可靠性的角度来考虑吗？

nsqd 写入消息、消费消息。

* 先说明整个消息从发布到消费的整个流程。

1. 启动nsqd ./nsqd --mem-queue-size=0  
2. 创建一个Topic curl -X POST http://127.0.0.1:4151/topic/create\?topic\=lmj
3. 往这个Topic里面发消息，curl -d "This is a Message" http://127.0.0.1:4151/pub\?topic\=lmj
4. 这个时候启动一个消费者，看看能不能消费到刚刚接收到的信息。

然后再把mem-queue-size调大。


nsqd 保证消息可靠性。
nsqd 和 nsqloopup 直接的交互
nsqd 集群相关

暂时就先从这几项开始吧。如果遇到别的内容再继续往上加。

下载源码，这个需要把源码放入到 $GOPATH/src/github.com/nsq-io/ 目录下。以防止产生引用错误。

#### reference
* NSQ源码 https://github.com/nsqio/nsq
