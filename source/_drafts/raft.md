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

在Raft协议中，节点有如下三种角色:
* Leader: 负责处理客户端请求、向Follower同步日志。
* Follower: 类似选民，完全被动，从Leader处获取日志。
* Candidate: Leader候选人。

![raft 状态转换](https://s2.ax1x.com/2019/01/09/FLlujg.png)

状态转换：

* 所有节点初始角色都是Follower。
* 如果Follower在超时时间内没有收到Leader的请求则转换为Candidate进行选举。
* Candidate收到大多数节点的选票则转换为Leader；发现Leader或者收到更高任期的请求则转换为Follower；如果出现多个Candidate票数相同的情况，则本次选举timeout，一段时间后开始下一个任期的选举。
* Leader在收到更高任期的请求后转换为Follower。

![raft 时间片](https://s2.ax1x.com/2019/01/09/FLlMuQ.png)

任期:

Raft把时间切割为任意长度的任期，每个任期都有一个任期号，采用连续的整数来进行表示。每个任期都由一次选举开始，若选举失败则这个任期内没有Leader；如果选举出了Leader则这个任期内有Leader负责集群状态管理。

### Leader 选举

一个 candidate 成为 leader 需要具备三个要素：
* 获得集群多数节点的同意；
* 集群中不存在比自己任期号更高的 candidate；
* 集群中不存在其他 leader。

#### 正常选举Leader的过程

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
