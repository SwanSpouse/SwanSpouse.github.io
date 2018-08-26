---
title: golang 内存管理提纲
date: 2018-08-22 23:57:25
tags:
  - golang
  - gc
---

这一阶段一直在看golang内存管理的相关内容，为了能让自己学习相关内容的思路更加清晰，参考jvm讲解内存管理的章节次序，列出了golang内存管理应当掌握的内容。

1. [golang new和make](https://swanspouse.github.io/2018/08/21/golang-new-object/) : 简单了解golang创建对象的基本方法。

2. [golang 逃逸分析](https://swanspouse.github.io/2018/08/21/golang-escape-analysis/) : 解决了对象是分配在堆上还是栈上的问题。

3. [tcmalloc](https://swanspouse.github.io/2018/08/17/tcmalloc/) : golang 内存管理算法的思想来源。

4. [golang 栈内存管理](https://swanspouse.github.io/2018/08/23/golang-stack/): 栈内存管理的主要数据结构，各个数据结构的作用以及分配算法。

5. [golang 堆内存管理、内存分配算法](https://swanspouse.github.io/2018/08/22/golang-memory-model/) : golang内存管理的主要数据结构，各个主要数据结构的作用以及针对不同的对象，采用的不同分配算法。

6. [对象存活性判断](https://swanspouse.github.io/2018/08/25/golang-gc-object/): 常见的判断对象是否存活的方法

7. [常见垃圾收集算法以及golang中的垃圾回收算法](https://swanspouse.github.io/2018/08/25/golang-gc-algorithm/): 对比Java Jvm内存回收。

8. [golang gc 迭代](https://swanspouse.github.io/2018/08/25/golang-gc-iterations/)golang各个版本垃圾回收机制的变化、改进
