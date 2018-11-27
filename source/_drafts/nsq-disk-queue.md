---
title: disk queue 源码分析
tags:
  - NSQ
---

### disk queue 简介

disk queue是基于文件存储的FIFO(first-in-first-out)队列。在对数据进行持久化的同时，保证数据写入和读出的相对顺序。

在NSQ中，disk queue的应用场景为：

* 生产者在生产消息时，当消息内存缓冲区已经满了的时候，则利用disk queue将消息存储在文件系统中。
* 消费者在进行消费时，如果内存缓冲区的数据已经被消费完，则利用disk queue来消费存储在文件系统中的消息。

### disk queue 实现方法

![diskQueue](https://s1.ax1x.com/2018/11/27/FEw1uF.png)

首先说明一下disk queue用于数据读写的几个变量：

* readFileNum : 表示当前应该从readFileNum这个序号的文件中读取数据。
* readPos :     表示当前应该从readFileNum这个序号的文件的readPos位置开始读取数据。
* writePosNum : 表示当前应该向writeFile这个序号的文件中写入数据。
* writePos :    表示当前应该从writeFileNum这个序号的文件的writePos位置开始写入数据。

在读取数据时，首先会根据readFileNum和readPos来确定应该从哪个文件的哪个位置开始读取数据；message数据的结构如上图所示，首先要读取4byte长度的数据，作为消息体的长度。在得知消息体长度之后，再读取相应byte的消息体。

在读取一条消息后，readPos和readFileNum指针会根据读取的字节数向后移动，若超出了单个文件的最大长度，那么直接去下一个新的数据文件中读取数据。

同理，写入消息时会根据writeFileNum和writePos来确定应该从哪个文件的哪个位置开始写入数据；确定位置后，先写入4byte数据作为消息长度，再写入消息体本身数据。写完当前消息之后，会向后挪动writePos和writeFileNum指针，同样的，超出单个文件的最大长度后，会在下次写入的时候创建一个新的数据文件，并将消息写入新的文件中。

### disk queue 主要结构

```golang
// diskQueue implements a filesystem backed FIFO queue
type diskQueue struct {
	// 64bit atomic vars need to be first for proper alignment on 32bit platforms

	// run-time state (also persisted to disk)
	readPos      int64 // diskQueue实例读取到readFileNum文件的位置
	writePos     int64 // diskQueue实例写入到writeFileNum文件的位置
	readFileNum  int64 // diskQueue实例当前应该读取的文件编号
	writeFileNum int64 // diskQueue实例当前应该写入的文件编号
	depth        int64 // 当前diskQueue中存储着多少条消息。相当于diskQueue的size

	sync.RWMutex // 对diskQueue操作需要加的读写锁

	// instantiation time metadata
	name            string // diskQueue实例的名称，用于作为路径名的标志
	dataPath        string // 文件存储路径
	maxBytesPerFile int64  // 每个存储文件的最大存储大小。一旦设定之后就不可以被更改。currently this cannot change once created
	minMsgSize      int32  // message 的最小值
	maxMsgSize      int32  // message 的最大值
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

### disk queue 主要方法


#### readOnce


#### writeOnce

#### ioLoop

#### reference

* disk queue source code : https://github.com/nsqio/go-diskqueue
