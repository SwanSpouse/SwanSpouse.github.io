---
title: golang逃逸分析
date: 2018-08-21 23:09:49
tags:
  - golang
  - gc
---

#### 逃逸分析

在编程语言的编译优化原理中，分析指针动态范围的方法称之为逃逸分析。通俗来讲，当一个对象的指针被多个方法或线程引用时，我们称这个指针发生了逃逸。发生逃逸行为的情况主要有两种：

* 方法逃逸：当一个对象在方法中定义之后，作为参数传递到其它方法中；
* 线程逃逸：如类变量或实例变量，可能被其它线程访问到；

#### 逃逸分析的优点

* 最大的好处应该是减少gc的压力，不逃逸的对象分配在栈上，当函数返回时就回收了资源，不需要gc标记清除。
* 逃逸分析完后可以确定哪些变量可以分配在栈上，栈的分配比堆快，性能好。
* 同步消除，如果你定义的对象的方法上有同步锁，但在运行时，却只有一个线程在访问，此时逃逸分析后的机器码，会去掉同步锁运行。

#### golang逃逸分析

go在一定程度消除了堆和栈的区别，因为go在编译的时候进行逃逸分析，来决定一个对象放栈上还是放堆上，不逃逸的对象放栈上，可能逃逸的放堆上。

在官网 (golang.org) FAQ 上有一个关于变量分配的问题如下：

How do I know whether a variable is allocated on the heap or the stack?

* From a correctness standpoint, you don’t need to know. Each variable in Go exists as long as there are references to it. The storage location chosen by the implementation is irrelevant to the semantics of the language.
* The storage location does have an effect on writing efficient programs. When possible, the Go compilers will allocate variables that are local to a function in that function’s stack frame. However, if the compiler cannot prove that the variable is not referenced after the function returns, then the compiler must allocate the variable on the garbage-collected heap to avoid dangling pointer errors. Also, if a local variable is very large, it might make more sense to store it on the heap rather than the stack.
* In the current compilers, if a variable has its address taken, that variable is a candidate for allocation on the heap. However, a basic_escape analysis_recognizes some cases when such variables will not live past the return from the function and can reside on the stack.

参考网上的翻译:

* 准确地说，你并不需要知道。Golang 中的变量只要被引用就一直会存活，存储在堆上还是栈上由内部实现决定而和具体的语法没有关系。
* 知道变量的存储位置确实和效率编程有关系。如果可能，Golang 编译器会将函数的局部变量分配到函数栈帧（stack frame）上。然而，如果编译器不能确保变量在函数return之后不再被引用，编译器就会将变量分配到堆上。而且，如果一个局部变量非常大，那么它也应该被分配到堆上而不是栈上。
* 当前情况下，如果一个变量被取地址，那么它就有可能被分配到堆上。然而，还要对这些变量做逃逸分析，如果函数return之后，变量不再被引用，则将其分配到栈上。

#### 逃逸分析的缺点

由于逃逸分析通常比较耗时，且性能未必能够提升多少。所以目前的实现都是采用不那么准确但是时间压力相对较小的算法来完成逃逸分析，这就可能导致效果不稳定，要慎用。


#### reference
