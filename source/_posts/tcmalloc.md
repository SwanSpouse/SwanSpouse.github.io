---
title: tcmalloc简介
date: 2018-08-17 22:48:53
tags:
  - golang
  - golang 内存管理模块
---

### 系统内存管理

内存管理可以分为三个层次，自底向上分别是：

* 操作系统层级：通过调用操作系统API来申请内存，如在Linux中通过brk，mmap，munmap这些系统调用来申请内存。
* 语言层级：例如glibc层使用系统调用维护的内存管理算法。也可以理解为包装好的，用作内存管理的三方库。
* 应用层级：应用程序从glibc动态分配内存后，根据应用程序本身的程序特性进行优化， 比如使用引用计数std::shared_ptr，apache的内存池方式等等。

其中，应用程序也可以直接通过系统调用来申请内存，并根据自己的程序特性对内存进行维护。但是会大大增加开发的成本。所以一般程序在开发的时候，都会选择使用三方库来进行内存管理。

### tcmalloc简介

tcmalloc(thread-caching malloc)在就是上面描述的语言层级中，由Google开源的内存管理库。其他常见的内存管理库还有glibc的ptmalloc和google的jemalloc。相比于ptmalloc，tcmalloc性能更好，特别适用于高并发场景。 作为glibc malloc的替代品。目前已经在chrome、safari等知名软件中运用。

根据官方测试报告，ptmalloc在一台2.8GHz的P4机器上（对于小对象）执行一次malloc及free大约需要300纳秒。而tcmalloc的版本同样的操作大约只需要50纳秒。

### tcmalloc思想

tcmalloc的基本思想是线程私有缓存和全局缓存。

线程私有缓存和全局缓存主要有以下两点不同：

* 线程私有性：每个线程都有自己的私有缓存，理想情况下，每个线程的内存管理都在自身的私有缓存中进行。没有了线程之间的竞争，所以效率非常高；

* 内存分配粒度：在tcmalloc里面，线程私有缓存和全局缓存都有两种内存粒度：object和span。span是连续内存page，而object则是由span切成的小块。object的尺寸被预设了一些规格（class），比如16Byte、32Byte等等，同一个span切出来的object都是相同的规格。

### tcmalloc分配策略

tcmalloc 在对于small object(<32KB)和 big object(>=32KB)的分配上采取了不同的分配策略。

#### small object allocation

![ThreadCache](https://s1.imgsha.com/2018/08/20/1AenW4.gif)

在小对象分配时，首先会在线程私有缓存中根据对象的大小来确定需要分配的object class，接着在相应class的 free object list中寻找第一个不为空的object进行分配；

若发现没有不为空的object，则会向全局缓存进行申请一批同样大小的object，将他们放到线程私有缓存中，然后再进行分配。


#### big object allocation

![big object allocation](https://s1.imgsha.com/2018/08/20/1Ae9hf.gif)

大对象直接在全局缓存中来进行分配。全局缓存的结构如上图所示，为单链表数组。数组长度为256，分别对应1 page大小(4KB), 2 page大小(8KB).... 最后一个对应>=256page的大小。

在分配时，首先计算大对象需要的page，假如需要K个page，则会在数组的第K个free list中寻找满足条件的内存，若没有满足条件的内存，则会在第K+1个free list中继续寻找，如果一直找到256都没有合适的内存，则会向操作系统申请内存。

对于在第 K+N个 free list中找到的满足条件的内存，分配后剩下的pages，会被放入到相应的free list中去。

### reference

* http://goog-perftools.sourceforge.net/doc/tcmalloc.html
* https://blog.csdn.net/junlon2006/article/details/77854898
