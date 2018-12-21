---
title: 'redis master, slave, sentinel通信'
tags:
  - redis
date: 2018-12-21 18:29:49
---


在这里主要说明redis sentinel是怎样在得知整个系统的拓扑结构的。

![redis sentinel](https://s1.ax1x.com/2018/12/21/FsYOG8.png)

1. 首先，sentinel会根据配置，和需要监控的master节点建立TCP连接，同时订阅master节点的 "\_\_sentinel\_\_:hello"频道。

2. 在建立TCP连接后， sentinel会每10s向master节点发送一次INFO命令。通过INFO命令的结果，sentinel可以得知master的信息，以及master下面所有slaves的信息。并把master和slaves的信息更新到本地。

3. 根据上一步骤的结果，sentinel会建立和slaves之间的TCP连接，同时订阅slave节点的 "\_\_sentinel\_\_:hello"频道。

4. 同样的，sentinel会每10s向slave节点发送INFO信息，同时将本地结果进行更新。

5. 同时在默认情况下，sentinel 会以每2s一次的频率向master和slaves节点PUBLISH 消息。这样所有监控了订阅了相应频道的sentinel就会收到各个sentinel发送的消息。

6. 订阅了"\_\_sentinel\_\_:hello" 频道的各个sentinel 会收到来自其他sentinel的消息。

7. sentinel 会建立和其他sentinel之间的TCP连接。至此。在程序、网络正常的情况下， master，slave和 sentinel 之间的连接就建立好了。

#### reference

* 《Redis 设计与实现》
