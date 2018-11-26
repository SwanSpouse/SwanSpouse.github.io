---
title: NSQ 简介
tags:
  - NSQ
date: 2018-11-26 22:54:00
---


NSQ 是实时的分布式消息处理平台，其设计的目的是用来大规模地处理每天数以十亿计级别的消息。

NSQ 具有分布式和去中心化拓扑结构，该结构具有无单点故障、故障容错、高可用性以及能够保证消息的可靠传递的特征。

### 基本组件

nsq 有三个必要的组建nsqd、nsqlookupd、nsqadmin。

#### nsqd

负责接收消息，存储队列和将消息发送给客户端。

nsqd 可以多机器部署，当客户端向指定topic发送消息时，可以配置多个nsqd地址，消息会随机的分配到各个nsqd上，nsqd优先把消息存储到内存channel中，当内存channel满了之后，则把消息写到磁盘文件中。


#### nsqlookupd

主要负责管理拓普信息、nsqd的心跳、状态监测，给客户端、nsqadmin提供topic所在的 nsqd地址与状态。

#### nsqadmin

nsqadmin是一个web管理界面，用来汇集集群的实时统计，并执行不同的管理任务。

#### message

消息，生产者与消费者之间传递的数据，在nsq中统一称为message。

#### topic

可以理解为生产者生产message的去处。当生产者生产message的时候，若没有相应的topic，则会在第一次生产message的时候创建Topic。

生产者和Topic是一对一的关系。

#### channel

消费者在消费message的时候，需要指定topic和channel。一个topic下可以有多个channel。channel会在消费者第一次订阅相应topic的时候就创建。

* 同一topic、同一channel下的多个消费者，不会消费到同一message。可以理解为多个消费者之间进行了负载均衡。

* 同一topic、不同channel下的两个消费者，可以消费到一模一样的message。 多个消费者之间相互独立，完全没有影响。

### 源码目录结构

```xml
nsq
├── apps                # 所有组件的main函数入口目录
│   ├── nsq_pubsub
│   ├── nsq_stat
│   ├── nsq_tail
│   ├── nsq_to_file
│   ├── nsq_to_http
│   ├── nsq_to_nsq
│   ├── nsqadmin        # nsqadmin 入口
│   ├── nsqd            # nsqd 入口
│   ├── nsqlookupd      # nsqlookup 入口
│   └── to_nsq
├── bench      # 批量测试工具
│   ├── bench.py
│   ├── bench_channels  #
│   ├── bench_reader    # 消息的消费者
│   ├── bench_writer    # 消息的生产者
├── contrib
│   ├── nsq.spec               # 可根据该文件生成nsq的rpm包
│   ├── nsqadmin.cfg.example   # nsqadmin配置文件举例
│   ├── nsqd.cfg.example       # nsqd配置文件举例
│   └── nsqlookupd.cfg.example # nsqlookup配置文件举例
├── internal    # nsq的基础库
├── nsqadmin    # web组件
├── nsqd        # nsqd具体实现，消息处理组件
│   ├── backend_queue.go       # nsqd 定义后台存储队列接口
│   ├── buffer_pool.go         # 缓存池
│   ├── channel.go             # channel 定义以及相关方法
│   ├── channel_test.go        # channel 相关测试
│   ├── client_v2.go           # client 定义 v2版本
│   ├── context.go             # topic、channel用于存储nsqd实例
│   ├── dqname.go              # backend names <topic>:<channel>格式
│   ├── dqname_windows.go      # backend names 用于windows系统，<topic>;<channel>格式
│   ├── dummy_backend_queue.go # 伪后台存储队列实现，所有需要落盘的数据全部丢弃。
│   ├── guid.go                # 生成全局唯一的guid
│   ├── guid_test.go           # 生成全局唯一的guid
│   ├── http.go                # 定义所有nsqd进程处理的http接口，以及相应的处理函数
│   ├── http_test.go           # http.go 相应的测试
│   ├── in_flight_pqueue.go    # 正在处理中的消息优先级队列。
│   ├── in_flight_pqueue_test.go # in_flight_pqueue.go 相应的测试
│   ├── logger.go              # log工具
│   ├── lookup.go              # 和 nsqlookupd 进行交互
│   ├── lookup_peer.go         # 和 nsqlookupd 进行交互的低层函数。读取、解析、发送命令等。
│   ├── message.go             # nsqd 传递消息的定义以及相关方法。
│   ├── nsqd.go                # nsqd 定义以及相关方法。
│   ├── nsqd_test.go           # nsqd.go 相关测试
│   ├── options.go             # nsqd 相关配置参数
│   ├── protocol_v2.go         # v2协议相关命令以及命令处理函数。
│   ├── protocol_v2_test.go    # protocol_v2.go 测试
│   ├── README.md         
│   ├── stats.go               # topic 和 channel 统计相关
│   ├── stats_test.go          # stats.go 相关测试
│   ├── statsd.go              # 和statsd 进行交互的相关内容
│   ├── tcp.go                 # 处理来自客户端的tcp连接
│   ├── topic.go               # topic 定义以及相关方法
│   ├── topic_test.go          # topic.go 相关测试
├── nsqlookupd      # 管理nsqd拓扑信息组件

```

#### reference

* nsq source code: https://github.com/nsqio/nsq

* https://juejin.im/entry/59ddae8151882578bb480d0e
