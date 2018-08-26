---
title: golang 栈内存管理
date: 2018-08-23 00:12:25
tags:
  - golang
  - gc
---

### golang 栈内存介绍

golang中栈内存用来存储函数内变量、执行函数调用等。在golang函数、协程退出后，其占用的栈内存也会一同被释放。

实际应用中协程的启动、退出可能会比较频繁，runtime必须要做点什么来保证启动、销毁协程的代价尽量小。而申请、释放stack空间所需内存则是一个比较大的开销，因此，go runtime采用了stack cache pool来缓存一定数量的stack memory。申请时从stack cache pool分配，释放时首先归还给stack cache pool。

### 核心数据结构

golang 栈内存的主要数据结构:

* mcache: mcache中的stackcache管理不同规格(class)的mspan：规格相同的mspan被链接到同一个链表中。
* stackpool：全局stack cache, 和mcache中的stackcache结构相同。
* stackLarge：全局stack cache, 和mcache中的stackcache结构相同。不同的是stackLarge中stack内存的规格。

其中stackcache的结构如下图所示:

![stack cache](https://s1.ax1x.com/2018/08/25/PbiuXd.png)

### 栈内存分配算法

栈内存管理的核心思想和堆内存很像。在分配时，首先查找线程内stackcache是否有足够的空间，如果有足够的空间，则进行分配，避免了线程间竞争，提高了效率；若线程内stackcache内存不足，则会向全局stackpool中申请一批stack，按照规格进行切分后，放入到线程的stackcache中，然后再次进行分配。

#### 普通栈内存分配(<32KB)

如上所述，首先会进行一些检查，找到合适的mspan规格、检查是否开启了stackCache、gc状态等。然后先检查本地stackcache相应的mspan list是否为空，如果为空，则会向全局的stackpoo申请一批stack，然后进行分配。

```golang
    order := uint8(0)
		n2 := n
		for n2 > _FixedStack {
      // 找到合适的规格class
			order++
			n2 >>= 1
		}
		var x gclinkptr
		c := thisg.m.mcache
		// 检查是否禁用了stackCache
		if stackNoCache != 0 || c == nil || thisg.m.preemptoff != "" || thisg.m.helpgc != 0 {
			// c == nil can happen in the guts of exitsyscall or
			// procresize. Just get a stack from the global pool.
			// Also don't touch stackcache during gc
			// as it's flushed concurrently.
			lock(&stackpoolmu)
			// 从全局的stack pool来进行分配
			x = stackpoolalloc(order)
			unlock(&stackpoolmu)
		} else {
			// 先在mcache中的stackcache中进行分配
			x = c.stackcache[order].list
			// 如果mcache中没有可以分配的内存
			if x.ptr() == nil {
				// 先从global cache pool中获取一批stack内存，
				// 将这批stack内存放入到mcache的stackcache中，
				stackcacherefill(c, order)
				x = c.stackcache[order].list
			}
      // 然后再从mcache中的stackcache进行获取
			c.stackcache[order].list = x.ptr().next
			c.stackcache[order].size -= uintptr(n)
		}
		v = unsafe.Pointer(x)
```

#### 大块栈内存分配(>=32KB)

对于大块栈内存，会直接在全局的stackLarge上进行分配，如果全局的stackLarge内存不足，则会向mHeap申请内存，然后再进行分配。

```golang

    var s *mspan
		npage := uintptr(n) >> _PageShift
		log2npage := stacklog2(npage)
		// Try to get a stack from the large stack cache.
		// 分配大块的stack
		lock(&stackLarge.lock)
		if !stackLarge.free[log2npage].isEmpty() {
			s = stackLarge.free[log2npage].first
			stackLarge.free[log2npage].remove(s)
		}
		unlock(&stackLarge.lock)

		if s == nil {
			// Allocate a new stack from the heap.
			// 如果global stack cache中还是没有申请到内存，那么会直接向mheap进行申请。
			s = mheap_.allocManual(npage, &memstats.stacks_inuse)
			if s == nil {
				throw("out of memory")
			}
			s.elemsize = uintptr(n)
		}
		v = unsafe.Pointer(s.base())

```

### reference

* golang soruce code 1.10
* https://tracymacding.gitbooks.io/implementation-of-golang/content/memory/memory_stack.html
