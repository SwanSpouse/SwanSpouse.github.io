---
title: golang内存模型
date: 2018-08-22 00:03:03
tags:
  - golang
---

### 内管管理

对于内置runtime system的编程语言，通常会抛弃传统的内存分配方式，改为自主管理内存。这样可以完成类似预分配、内存池、垃圾回收等操作，以避开频繁地向操作系统申请、释放内存，产生过多的系统调用而导致的性能问题。

golang的runtime system同样实现了一套内存池机制，接管了所有的内存申请和释放的动作。

### 核心数据结构

mheap: mheap管理向os申请、释放、组织mspan；
mcentral: mcentral按照自己管理的块大小将mspan分配给mcache；
mspan: mspan是数据的实际存储区域，按照mcentral管理的块规格(class)被切分成小块。
mcache: mcache管理不同规格(class)的mspan：规格相同的mspan被链接到同一个链表中。

![go heap model](https://s1.imgsha.com/2018/08/22/1aPAcC.png)

![mcache](https://s1.imgsha.com/2018/08/22/1aPKq2.png)

### 内存分配算法

为提高内存分配的效率，针对不同的对象大小，golang runtime system采用了不同的分配策略。

#### tiny object allocation

对于小于maxTinySize(16B)字节对象的内存分配请求。go采取了将小对象合并存储的解决方案。在线程的本地缓存中维护了专门的区域来存储tiny object。

#### big object allocation

#### small object allocation

### reference
* https://tracymacding.gitbooks.io/implementation-of-golang/content/
* http://legendtkl.com/2017/04/02/golang-alloc/
