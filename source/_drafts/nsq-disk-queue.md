---
title: disk queue 源码分析
tags:
  - NSQ
---

#### disk queue 简介

disk queue是基于文件存储的FIFO(first-in-first-out)队列。在对数据进行持久化的同时，保证数据写入和读出的相对顺序。

在NSQ中，disk queue的应用场景为：

* 生产者在生产消息时，当消息内存缓冲区已经满了的时候，则利用disk queue将消息存储在文件系统中。
* 消费者在进行消费时，如果内存缓冲区的数据已经被消费完，则利用disk queue来消费存储在文件系统中的消息。

#### disk queue 主要结构

```golang
// diskQueue implements a filesystem backed FIFO queue
type diskQueue struct {
	// 64bit atomic vars need to be first for proper alignment on 32bit platforms

	// run-time state (also persisted to disk)
	readPos      int64 // diskQueue实例读取到readFileNum文件的位置
	writePos     int64 // diskQueue实例写入到writeFileNum文件的位置
	readFileNum  int64 // diskQueue实例当前应该读取的文件编号
	writeFileNum int64 // diskQueue实例当前应该写入的文件编号
	depth        int64

	sync.RWMutex // 对diskQueue操作需要加的读写锁

	// instantiation time metadata
	name            string // diskQueue实例的名称，用于作为路径名的标志
	dataPath        string // 文件存储路径
	maxBytesPerFile int64  // 每个存储文件的最大存储大小。一旦设定之后就不可以被更改。currently this cannot change once created
	minMsgSize      int32
	maxMsgSize      int32
	syncEvery       int64         // number of writes per fsync
	syncTimeout     time.Duration // duration of time per fsync
	exitFlag        int32
	needSync        bool // 标志是否需要进行同步

	// keeps track of the position where we have read
	// (but not yet sent over readChan)
	nextReadPos     int64 // diskQueue实例下次应该读取的文件位置。
	nextReadFileNum int64 // diskQueue实例下次应该读取的文件编号

	readFile  *os.File      // 当前读取的文件
	writeFile *os.File      // 当前写入的文件
	reader    *bufio.Reader // 当前读取文件的reader
	writeBuf  bytes.Buffer  // 当前写入文件的writer

	// exposed via ReadChan()
	readChan chan []byte // 用于接受需要读取的byte数组

	// internal channels
	writeChan         chan []byte // 用于接受需要写入的byte数组
	writeResponseChan chan error  // 对“写”操作的回复
	emptyChan         chan int    // 用于接受“清空”操作的信号
	emptyResponseChan chan error  // 对“清空”操作的回复
	exitChan          chan int    // 用于标志是否退出
	exitSyncChan      chan int

	logf AppLogFunc // log 函数
}
```

#### reference

* disk queue source code : https://github.com/nsqio/go-diskqueue
