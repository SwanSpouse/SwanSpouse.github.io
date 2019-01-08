---
title: raft协议
tags:
    - distributed system
    - raft
---


### raft协议简介

Raft协议是用于解决分布式系统中多副本之间的数据一致性问题。

Raft算法的优点在于可以在高效的解决分布式系统中各个节点日志内容一致性问题的同时，也使得集群具备一定的容错能力。即使集群中出现部分节点故障、网络故障等问题，仍可保证其余大多数节点正确的步进。甚至当更多的节点（一般来说超过集群节点总数的一半）出现故障而导致集群不可用时，依然可以保证节点中的数据不会出现错误的结果。

![raft处理客户端请求](https://s2.ax1x.com/2019/01/09/FLlhbd.png)

#### raft 基本概念

Raft 集群中的每个节点都可以根据集群运行的情况在三种状态间切换：follower, candidate 与 leader。leader 向 follower 同步日志，follower 只从 leader 处获取日志。在节点初始启动时，节点的 raft 状态机将处于 follower 状态并被设定一个 election timeout，如果在这一时间周期内没有收到来自 leader 的 heartbeat，节点将发起选举：节点在将自己的状态切换为 candidate 之后，向集群中其它 follower 节点发送请求，询问其是否选举自己成为 leader。当收到来自集群中过半数节点的接受投票后，节点即成为 leader，开始接收保存 client 的数据并向其它的 follower 节点同步日志。leader 节点依靠定时向 follower 发送 heartbeat 来保持其地位。任何时候如果其它 follower 在 election timeout 期间都没有收到来自 leader 的 heartbeat，同样会将自己的状态切换为 candidate 并发起选举。每成功选举一次，新 leader 的步进数都会比之前 leader 的步进数大 1。

在Raft协议中，节点有如下三种角色:
* Leader: 负责处理客户端请求、日志复制等。
* Follower: 类似选民，完全被动。
* Candidate: Leader候选人。

![raft 状态转换](https://s2.ax1x.com/2019/01/09/FLlujg.png)

![raft 时间片](https://s2.ax1x.com/2019/01/09/FLlMuQ.png)

#### Leader election

正常情况下选举Leader

一个 candidate 成为 leader 需要具备三个要素：
获得集群多数节点的同意；
集群中不存在比自己步进数更高的 candidate；
集群中不存在其他 leader。


Leader出故障的时候重新选举Leader

出现多个Candidate情况下选举Leader

多个 candidate 或多个 leader

新节点加入集群

#### Log replication

#### Safety

#### reference

* In Search of an Understandable Consensus Algorithm https://raft.github.io/raft.pdf
* The Raft Consensus Algorithm https://raft.github.io/
* raft协议动画演示 http://thesecretlivesofdata.com/raft/
* https://www.jianshu.com/p/8e4bbe7e276c
* https://www.infoq.cn/article/coreos-analyse-etcd
