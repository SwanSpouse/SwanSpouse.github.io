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

对于小于maxTinySize(16B)字节对象的内存分配请求。go采取了将小对象合并存储的解决方案。在线程的本地缓存中维护了专门的区域(mcache.tiny)来存储tiny object。

在请求tiny object内存分配的时候，首先查看mcache.tiny的剩余空间是否能够满足tiny object对象的分配。如果足够则直接返回；如果mcache.tiny的内存不够分配，则需要向上一届mcache, mcentral, mheap, system依次申请内存，然后再分配。

```golang

if noscan && size < maxTinySize {
	off := c.tinyoffset
	// Align tiny pointer for required (conservative) alignment.
	// 实际上占用的内存地址会按照一定的规则进行对齐
	//   1. 如果大于等于8B，按照8B对齐；
	//   2. 如果大于4B小于8B，按照4B对齐；
	//   3. 如果大于1B小于4B，按照2B对齐；
	//   4. 对于1B对象，无对齐要求； TODO lmj 其实没有太理解这里为啥能够按照上面说的方式进行对齐。
	if size&7 == 0 {
		off = round(off, 8)
	} else if size&3 == 0 {
		off = round(off, 4)
	} else if size&1 == 0 {
		off = round(off, 2)
	}
	/* 在c.tiny!=0且mcache中已分配tiny object内存+要分配size <= maxTinySize的时候，
	直接在mcache分配。
	*/
	if off+size <= maxTinySize && c.tiny != 0 {
		// The object fits into existing tiny block.
		// x就是分配出去的内存块的首地址
		x = unsafe.Pointer(c.tiny + off)
		// 将c.tinyoffset向后移，表示size已被分配
		c.tinyoffset = off + size
		c.local_tinyallocs++
		mp.mallocing = 0
		releasem(mp)
		return x
	}
	// Allocate a new maxTinySize block.
	/*
	由于mcache是一个数组+链表的结构。所以这一步是直接根据spanClass(span的规格)获取到数组中链表的头指针。
	也就是符合规格span中打头的第一个。然后沿着个span遍历链表，找到第一个符合条件的span。
	 */
	span := c.alloc[tinySpanClass]
	// 这里尝试在mcache中获取，也就是从span开始遍历链表
	v := nextFreeFast(span)
	// v == 0 说明在链表中没有找到合适规格的span
	if v == 0 {
		// 如果第一次没有获取到，那么则尝试向mcentral来进行获取
		v, _, shouldhelpgc = c.nextFree(tinySpanClass)
	}
	// 分配内存的首地址
	x = unsafe.Pointer(v)
	// 把地址置零值 64/8 = 8B，两个8B=16B，所以现在每申请一个tinySpan都是一个16B的内存
	(*[2]uint64)(x)[0] = 0
	(*[2]uint64)(x)[1] = 0
	// See if we need to replace the existing tiny block with the new one
	// based on amount of remaining free space.
	// 根据剩余的free space，判断是否需要用新的tiny block来替代老的tiny block
	if size < c.tinyoffset || c.tiny == 0 {
		c.tiny = uintptr(x)
		c.tinyoffset = size
	}
	size = maxTinySize
}

```

#### big object allocation




#### small object allocation

### reference
* https://tracymacding.gitbooks.io/implementation-of-golang/content/
* http://legendtkl.com/2017/04/02/golang-alloc/
