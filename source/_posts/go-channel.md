---
title: 深入理解 go channel
tags:
  - go
date: 2018-10-15 23:39:26
---


### 概述

Channels provide a simple mechanism for goroutines to communicate, and a powerful construct to build sophisticated concurrency patterns. We will delve into the inner workings of channels and channel operations, including how they're supported by the runtime scheduler and memory management systems.

channel是goroutine之间的通信机制，同时，channel也是实现复杂高并发程序的基础。在这里会对channel内部的工作机制，包括channel如何被调度器调度、内存管理系统等进行深入的说明。

A fair number of us are using Go for concurrency features. Channels are not just useful, they are interesting.

只有少部分人使用go语言是因为它在并发方面的优势。channel不仅有用，而且还很有趣。

We can implement a simple task queue with just channels.

我们可以用channel实现一个简单的任务队列。

![task_queue](https://s1.ax1x.com/2018/10/15/ialATe.png)


Channels are interesting because:
  Channels are goroutine safe
  Channels carry messages between goroutines
  FIFO semantics
  Channels cause blocking and unblocking

Channel 很有趣，因为:
* channel 是goroutine安全的。
* channel 可以在goroutine 之间传递消息。
* 遵循FIFO语法
* channel 可以是阻塞或者非阻塞的。

Even though we understand all these properties, you take a second to appreciate and you want to understand how it works.

尽管我们知道所有的这些特定，你可以花一点时间来了解下它是怎么工作的。

This talk is about:

Making channels (the hchan struct)
Sends and receives (goroutine scheduling)
Stepping back (design considerations)

这篇文章主要包括：

* 创建channel（hchan结构）
* 发送和接受消息（goroutine 调度）
* 回顾（关于设计的思考）

### 创建 channel

First, you need to first create a channel to use it. make can create both buffered and unbuffered channels. This talk is primarily about buffered channels.

在使用之前首先要创建一个channel，make可以创建带缓冲区或者不带缓冲区的channel。这篇文章主要讨论的是带有缓冲区的channel。

A buffered channel with a capacity of 3: ch := make(chan Task, 3)

Goroutine safe
Stores up to capacity elements and provides FIFO semantics
Sends values between goroutines
Can cause block/unblock

创建一个缓冲区容量为3的channel可以使用 ch := make(chan Task, 3)

* 这个channel是goroutine安全的。
* 存储到channel中的数据是遵循FIFO的。
* 可以在goroutine之间传输消息。
* 能够使goroutine阻塞或者非阻塞。

![making channels](https://s1.ax1x.com/2018/10/15/ialdXV.png)

make chan allocates an hchan struct on the heap and initializes it and returns a pointer to it. This is why we can pass channels between functions.

make chan 在堆内存上创建了一个hchan结构，返回指向它的指针。这使得我们可以在方法之间传递channel。

#### 使用 channel

![using channel](https://s1.ax1x.com/2018/10/15/ial0mT.png)

How send and receive works?
G1 is the sender and G2 is the worker.
G1 comes along first and sends to the channel. The lock is acquired and the value is queued. Queuing is actually a memory copy.
Then, G2 comes along and does the inverse. It acquires the lock and dequeues which also mem copies the value.
Copying gives us the safeness, the channel is protected by mutex and nothing else is shared. Values are copied.

channel是怎样发送和接收消息的？
* G1发送任务，G2接收并执行任务。
* 首先，G1向channel中发送任务，将任务写入队列的这个过程是需要锁的。入队的值实际上是原值的一个拷贝。
* 接着，G2做相反的工作：将任务出队。这个过程也是需要锁的，同样的，出队的值实际上也是原值的一个拷贝。
* 每次操作时，操作的都是值的拷贝保证了goroutine安全，操作的过程channel是加锁的保证了互斥。

How blocking/unblocking works?
Assume G1 keep sending and it takes a long time to process G2. When channel is full, G1’s execution is paused; how does pausing work?

It happens at the runtime scheduler.
Goroutines are user-space threads, created and managed by the Go runtime, not the OS. These are lightweight compared to OS threads w.r.t. research consumption and scheduling overhead.
Go scheduler is a M:N scheduler: few OS threads and N goroutines; and scheduler is responsible to schedule the goroutines on limited number of OS threads.

channel的阻塞和非阻塞是怎样工作的？

假设G1不停的向channel中发送任务，同时G2处理任务需要花费一些时间。当channel满了的时候，G1就会暂停执行。G1是怎么暂停的呢?

它是由runtime 调度器来实现的。

Goroutine 是Go runtime创建和管理的用户级别线程，而不是由系统来管理的。这些线程和系统线程相比更加轻量级。

Go语言调度器是一个M：N的调度器，将少部分系统线程映射到N个goroutine上，调度器的任务就是调度goroutine，让其运行在少数的系统线程上。

#### 暂停 goroutine

![pausing goroutine](https://s1.ax1x.com/2018/10/15/ialjnf.png)

When a goroutine needs to be paused, chan calls into the scheduler to park G1, what scheduler does is to change G1 from running to waiting. And schedule another goroutine on the OS thread.

当goroutine需要暂停的时候，调度器在接收到chan的通知后，将G1从running状态置为waiting状态，同时调度其它goroutine使用空出来的系统线程。

This is good for perf. We haven’t stopped the OS thread but scheduled another goroutine by context switching. This is not expensive.
We need to resume the paused goroutine once the channel is not full anymore.

这个做法非常棒，因为我们并没有停止系统线程的运行，只是通过上下文切换的方式更换了正在运行的goroutine，这个操作的代价很小。

当channel又有存储空间的时候，我们需要将这个goroutine从暂停的状态上恢复回来。

#### 回复 goroutine

![resuming goroutine](https://s1.ax1x.com/2018/10/15/ia1SAg.png)

Waiting goroutine struct has a pointer to the element it is waiting on.
G1 created a sudog for itself, puts it in the channel’s wait queue. Sudog will be used to resume the G1 in the future.
What G2 does when the channel is not full anymore is to pop off the sudog. G1 is set runnable again. The runtime schedules G1 again and G1 resumes. (Also explains the case what happens when G2 needs to be parked and resumed again.)

处于等待状态的goroutine结构中有一个指针，指向它正在等待的元素？

G1自己创建了一个sudog，将它放入到channel的等待队列中，用于在以后恢复G1的运行。

当channel又存在存储空间的时候，G2需要将sudog出队，这时G1又会被恢复运行。runtime 调度器再一次调度G1，恢复G1的运行。(这也解释了G2被暂停和恢复的原因。)

#### 直接发送

![direct send](https://s1.ax1x.com/2018/10/15/ia1i3n.png)

When G1 finally runs, it needs to acquire the lock. But the runtime is actually is so much smarter to make this less costly. Runtime can copy directly to the receiver stack. G1 writes directly to G2’s stack and doesn’t have to acquire any locks.

On resuming, G2 does not need to acquire channel lock and manipulate the buffer. This also means one fewer memory copy.

当G1真正运行的时候，它首先要获取锁，但是runtime 更加机智，在此做了优化。runtime可以直接将值拷贝到接收者的栈内存。 G1直接写G2的栈内存，这个过程并不需要锁；G2在读取的时候也不再需要channel锁而直接去操作缓存，这样节省了值拷贝的内存空间。


### reference
* https://about.sourcegraph.com/go/understanding-channels-kavya-joshi/
