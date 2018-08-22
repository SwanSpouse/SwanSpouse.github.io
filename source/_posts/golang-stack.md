---
title: golang stack内存管理
date: 2018-08-23 00:12:25
tags:
  - golang
---

#### golang 栈内存介绍

golang中栈内存用来存储函数内变量、执行函数调用等。在golang函数、协程退出后，其占用的栈内存也会一同被释放。

实际应用中协程的启动、退出可能会比较频繁，runtime必须要做点什么来保证启动、销毁协程的代价尽量小。而申请、释放stack空间所需内存则是一个比较大的开销，因此，go runtime采用了stack cache pool来缓存一定数量的stack memory。申请时从stack cache pool分配，释放时首先归还给stack cache pool。



#### reference
