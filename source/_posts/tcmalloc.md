---
title: tcmalloc简介
date: 2018-08-17 22:48:53
tags:
  - golang
---

### 系统内存管理

内存管理可以分为三个层次，自底向上分别是：

* 操作系统层级：通过调用操作系统API来申请内存，如在Linux中通过brk，mmap，munmap这些系统调用来申请内存。
* 语言层级：例如glibc层使用系统调用维护的内存管理算法。也可以理解为包装好的，用作内存管理的三方库。
* 应用层级：应用程序从glibc动态分配内存后，根据应用程序本身的程序特性进行优化， 比如使用引用计数std::shared_ptr，apache的内存池方式等等。

其中，应用程序也可以直接通过系统调用来申请内存，并根据自己的程序特性对内存进行维护。但是会大大增加开发的成本。所以一般程序在开发的时候，都会选择使用三方库来进行内存管理。

### tcmalloc简介

tcmalloc 在就是上面描述的语言层级中，由Google开源的内存管理库。其他常见的内存管理库还有glibc的ptmalloc和google的jemalloc。相比于ptmalloc，tcmalloc性能更好，特别适用于高并发场景。 作为glibc malloc的替代品。目前已经在chrome、safari等知名软件中运用。

根据官方测试报告，ptmalloc在一台2.8GHz的P4机器上（对于小对象）执行一次malloc及free大约需要300纳秒。而tcmalloc的版本同样的操作大约只需要50纳秒。

### tcmalloc思想

tcmalloc的基本思想是多级内存管理。前面层次内存分配失败，则向后面层次申请；前面层次内存释放过多，则回收到后面层次。

### tcmalloc实现

如上述所说，在tcmalloc内存管理的体系之中，一共有三个层次：ThreadCache、CentralCache、PageHeap。

分配内存和释放内存的时候都是按从前到后的顺序，在各个层次中去进行尝试。基本思想是：前面的层次分配内存失败，则从下一层分配一批补充上来；前面的层次释放了过多的内存，则回收一批到下一层次。

这几个层次从前到后，主要有这么几方面的变化：

* 线程私有性：ThreadCache，顾名思义，是每个线程一份的。理想情况下，每个线程的内存需求都在自己的ThreadCache里面完成，线程之间不需要竞争，非常高效。而CentralCache和PageHeap则是全局的；

* 内存分配粒度：在tcmalloc里面，有两种粒度的内存，object和span。span是连续page的内存，而object则是由span切成的小块。object的尺寸被预设了一些规格（class），比如16字节、32字节、等等，同一个span切出来的object都是相同的规格。object不大于256K，超大的内存将直接分配span来使用。ThreadCache和CentralCache都是管理object，而PageHeap管理的是span。

#### ThreadCache

维护一组FreeList，针对每一种class的object；

#### CentralCache

里面有多个CentralFreeList，针对每一种class的object。

CentralFreeList并不像ThreadCache那样直接维护object的链表，而是维护span的链表，每个span下面再挂一个由这个span切分出来的object的链。这样做便于在span内的object是否都已经free的情况下，将span整体回收给PageHeap（span.refcount_记录了被分配出去的object个数）。但是这样一来，每个回收回来的object都需要寻找自己所属的span，然后才能挂进freelist，过程会比较耗时。

所以CentralFreeList里面还搞了一个cache（tc_slots_），回收回来的一批object先往cache里面塞，塞不下了再回收进span的objects链。分配object给ThreadCache时也是先尝试在cache里面拿，没了再去span里面分配。

另外，CentralFreeList里的span链表其实是有两个：nonempty_和empty_，根据span的objects链是否有空闲，放入对应链表。这样就避免了在分配时去判断span是否为空，只需要在由空变非空、或者由非空变空时移动一下span。

#### PageHeap
维护了两个很重要的东西：page到span的映射关系，和空闲span的伙伴系统。



### reference
* https://blog.csdn.net/junlon2006/article/details/77854898
* https://yq.aliyun.com/articles/6045
