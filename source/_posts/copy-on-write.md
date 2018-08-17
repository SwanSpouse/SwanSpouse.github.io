---
title: copy_on_write机制
date: 2018-08-17 17:43:53
tags:
  - java
---

#### copy_on_write 简介

Copy-On-Write简称COW，是一种惰性修改策略。其基本思路为：最开始的时候大家共享同一份内容。当有人需要对内容进行修改的时候，先将内容复制一份形成副本，并对复本进行修改。修改结束后，再用副本替代原内容。

#### CopyOnWrite 容器

从JDK1.5开始Java并发包里提供了两个使用CopyOnWrite机制实现的并发容器,
* CopyOnWriteArrayList
* CopyOnWriteArraySet

CopyOnWrite容器即写时复制的容器。通俗的理解是当我们往一个容器添加元素的时候，不直接往当前容器添加，而是先将当前容器进行复制，复制出一个新的容器，然后向新的容器里添加元素，添加完元素之后，再将原容器的引用指向新的容器。这样做的好处是我们可以对CopyOnWrite容器进行并发的读，而不需要加锁，因为当前容器不会添加任何元素。所以CopyOnWrite容器也是一种读写分离的思想，读和写不同的容器。

在执行写操作的时候，是需要进行加锁的。否则在多线程情况下会Copy出多个容器副本。

#### 应用场景

CopyOnWrite并发容器用于读多写少的并发场景。

#### CopyOnWrite的缺点

内容占用问题
* 因为CopyOnWrite的写时复制机制，所以在进行写操作的时候，内存里会同时驻扎两个容器对象。

数据一致性问题
* CopyOnWrite容器只能保证数据的最终一致性，不能保证数据的实时一致性。

#### reference
* http://ifeve.com/java-copy-on-write/
