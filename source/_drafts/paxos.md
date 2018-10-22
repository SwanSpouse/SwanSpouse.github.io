---
title:  Paxos 算法
tags:
  - distributed system
---


### Paxos 算法背景

Paxos算法是莱斯利·兰伯特（Leslie Lamport，就是 LaTeX 中的”La”，此人现在在微软研究院）于1990年提出的一种基于消息传递的一致性算法。由于算法难以理解起初并没有引起人们的重视，使Lamport在八年后重新发表到TOCS上[2]。即便如此paxos算法还是没有得到重视，2001年Lamport用可读性比较强的叙述性语言给出算法描述[3]。可见Lamport对paxos算法情有独钟。近几年paxos算法的普遍使用也证明它在分布式一致性算法中的重要地位。06年google的三篇论文初现“云”的端倪，其中的chubby锁服务使用paxos作为chubby cell中的一致性算法，paxos的人气从此一路狂飙。

Paxos 算法是分布式一致性算法用来解决一个分布式系统如何就某个值(决议)达成一致的问题。

一个典型的场景是，在一个分布式数据库系统中，如果各节点的初始状态一致，每个节点都执行相同的操作序列，那么他们最后能得到一个一致的状态。为保证每个节点执行相同的命令序列，需要在每一条指令上执行一个”一致性算法”以保证每个节点看到的指令一致。

#### 拜占庭问题

Paxos算法的使用范围。


### reference
* https://blog.csdn.net/xiaqunfeng123/article/details/51712983
