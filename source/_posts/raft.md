---
title: raft协议
tags:
    - distributed system
    - raft
---

### raft协议简介

Paxos一直是分布式协议的标准，但是Paxos难于理解，更难以实现，Google的分布式锁系统Chubby作为Paxos实现曾经遭遇到很多坑。

来自Stanford的新的分布式协议研究称为Raft，它是一个为真实世界应用建立的协议，主要注重协议的落地性和可理解性。

在Raft系统中，任何时候一个服务器可以扮演下面的角色之一：

* 领导者 Leader：处理所有客户端交互（读写操作）、日志复制等动作，要一个Raft Group只有一个Leader。
* 选民Follower：类似选民，完全被动的角色，这样的服务器等待被通知投票。
* 候选人Candidate：候选人就是在选举过程中提名自己的实体，一旦选举成功，则成为领导者。

### Raft 日志同步（复制）过程

1. 客户端向Leader发送写请求。
2. Leader接收到请求之后会把数据写入log中，此时还处于uncommitted状态。
3. 此时Leader会通过心跳将日志发送给其他Follower节点。
4. Follower节点写入log entry，并返回给Leader结果。Leader在收到大多数Follower的写入结果之后，会将log entry进行commit，并返回结果给客户端。
5. Leader再告知当前的log entry已经commit。Follower节点会进行commit。
6. 此时系统内部达成一致性。

### Raft Leader选举过程：

Raft election timeout: 这个timeout的含义是，当一个处于follower状态的节点等待这么长时间之后，它就会从Follower变成Candidate。各个节点的election timeout的时间是150ms - 300ms不等的随机数。

1. 最开始的时候，所有的节点都处于Follower状态。
2. 当一个节点进过election timeout的时间没有接收到来自Leader的心跳的时候，这个节点的状态会从Follower 切换成 Candidate，处于Candidate状态的节点会要求其它节点向其进行投票。（自己的票肯定是投给自己的）
3. 当处于Follower状态的节点收到来自其他节点的投票请求，并在当前周期内没有投过票的时候。会对当前Candidate进行投票。同时重置自己的election timeout计时器。
4. 当一个Candidate节点接收到大多数节点对自己的投票的时候，它就会从Candidate状态切换成Leader状态。
5. Leader会向Follower同步自己的消息。

### Raft Leader重新选举过程：

如果两个Candidate节点在同一个任期内，同时参与竞选。且两者得票数相同。那么在当前任期则不会有Leader产生。等待election time的时间后，会进行下一个任期的竞选。

Splite Vote是因为如果同时有两个Candidate向大家邀票，这时通过类似加时赛来解决，两个Candidate在一段timeout比如300ms互相不服气的等待以后，因为双方得到的票数是一样的，一半对一半，那么在300ms以后，再由这两个Candidate发出邀票，这时同时的概率大大降低，那么首先发出邀票的的Candidate得到了大多数同意，成为领导者Leader，而另外一个Candidate后来发出邀票时，那些Follower选民已经投票给第一个Candidate，不能再投票给它，它就成为落选者了，最后这个落选者也成为普通Follower一员了。

### 脑裂问题：

A B C D E 5个节点，假如当前B是Leader，其他的节点是Follower。现在A B 两个节点、C D E三个节点割裂成2个网络。

假设客户端连接的是A B网络，那么客户端在写入的时候，由于大多数节点无法进行commit所以会导致写入失败。

假设客户端连接的是C D E网络（此时C D E已经选举出新的Leader），那么客户端在写入的时候，由于存在3个节点，能够写入成功。

此时相当于A B网络的节点一直保持着原有的状态，没有任何写入。相当于一直等待着被连进网络。

当两个网络又重新被打通的时候，由于C D E的任期序号要高于 A B网络，所以 A B 节点中的数据会进行回滚，同时从Leader中重新拉取数据，此时，集群节点的状态重新达到一致。

#### reference

* In Search of an Understandable Consensus Algorithm https://raft.github.io/raft.pdf
* The Raft Consensus Algorithm https://raft.github.io/
* raft协议动画演示 http://thesecretlivesofdata.com/raft/
* https://www.jianshu.com/p/8e4bbe7e276c
* https://www.infoq.cn/article/coreos-analyse-etcd
