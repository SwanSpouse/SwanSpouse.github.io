---
title: nsq 源码解读提纲
tags:
    - NSQ
---

nsq 代码目录结构。
nsqd 相关解读。 topic channel 等等这些基本概念。
这里先说diskqueue吧。
nsqd 写入消息、消费消息。
nsqd 保证消息可靠性。
nsqd 和 nsqloopup 直接的交互
nsqd 集群相关

暂时就先从这几项开始吧。如果遇到别的内容再继续往上加。

下载源码，这个需要把源码放入到 $GOPATH/src/github.com/nsq-io/ 目录下。以防止产生引用错误。

#### reference
《nsq source code》
