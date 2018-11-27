---
title: disk queue 源码分析
tags:
  - NSQ
date: 2018-11-27 18:33:06
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

同理，写入消息时会根据writeFileNum和writePos来确定应该从哪个文件的哪个位置开始写入数据；确定位置后，先写入4byte数据作为消息长度，再写入消息数据。写完当前消息之后，会向后挪动writePos和writeFileNum指针，同样的，超出单个文件的最大长度后，会在下次写入的时候创建一个新的数据文件，并将消息写入新的文件中。

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

diskqueue.go 主要包括了3个主要的方法：readOnce, writeOnce, ioLoop。

#### readOnce

readOnce方法的作用为从磁盘中读取一条message。

```golang

// readOne performs a low level filesystem read for a single []byte
// while advancing read positions and rolling files, if necessary
// 根据规定的消息写入格式，从文件中读取一条消息
func (d *diskQueue) readOne() ([]byte, error) {
	var err error
	var msgSize int32

	// 两种情况这个readFile == nil
	// 	1. 当读完上一个文件的时候，会关闭掉上一个文件的句柄。并在下面打开应该读取的新文件。
	//	2. 当diskQueue启动的时候，会打开相应的数据文件，并查找到应该进行读的位置。
	if d.readFile == nil {
		curFileName := d.fileName(d.readFileNum)
		d.readFile, err = os.OpenFile(curFileName, os.O_RDONLY, 0600)
		if err != nil {
			return nil, err
		}
		d.logf(INFO, "DISKQUEUE(%s): readOne() opened %s", d.name, curFileName)

		// 找到应该读取的位置
		if d.readPos > 0 {
			_, err = d.readFile.Seek(d.readPos, 0)
			if err != nil {
				d.readFile.Close()
				d.readFile = nil
				return nil, err
			}
		}

		d.reader = bufio.NewReader(d.readFile)
	}
	// 首先读取4个字节的消息长度
	err = binary.Read(d.reader, binary.BigEndian, &msgSize)
	if err != nil {
		d.readFile.Close()
		d.readFile = nil
		return nil, err
	}
	// 验证消息长度是否合法
	if msgSize < d.minMsgSize || msgSize > d.maxMsgSize {
		// this file is corrupt and we have no reasonable guarantee on
		// where a new message should begin
		d.readFile.Close()
		d.readFile = nil
		return nil, fmt.Errorf("invalid message read size (%d)", msgSize)
	}
	// 根据消息的长度创建相应的readBuf，并去读msgSize个字节到readBuf中
	readBuf := make([]byte, msgSize)
	_, err = io.ReadFull(d.reader, readBuf)
	if err != nil {
		d.readFile.Close()
		d.readFile = nil
		return nil, err
	}
	// 本次总共读取的字节数为4个字节消息长度 + msgSize个字节的消息体
	totalBytes := int64(4 + msgSize)
	// we only advance next* because we have not yet sent this to consumers
	// (where readFileNum, readPos will actually be advanced)
	// 下一次应该读取数据的位置和文件号
	d.nextReadPos = d.readPos + totalBytes
	d.nextReadFileNum = d.readFileNum

	// TODO: each data file should embed the maxBytesPerFile
	// as the first 8 bytes (at creation time) ensuring that
	// the value can change without affecting runtime
	// 如果下一次应该读取的数据超过了单个文件的长度，则从下一个文件中进行读取
	if d.nextReadPos > d.maxBytesPerFile {
		if d.readFile != nil {
			d.readFile.Close()
			d.readFile = nil
		}

		d.nextReadFileNum++
		d.nextReadPos = 0
	}
	return readBuf, nil
}

```

#### writeOnce

writeOnce方法的作用为向磁盘中写入一条message。

```golang
// writeOne performs a low level filesystem write for a single []byte
// while advancing write positions and rolling files, if necessary
// 根据规定的消息写入格式，写入一条消息。
func (d *diskQueue) writeOne(data []byte) error {
	var err error
	// 两种情况这个writeFile == nil
	// 	1. 当上一个文件写满了的时候，会关闭掉上一个文件的句柄。在下面创建新的数据文件。
	//	2. 当diskQueue启动的时候，会打开相应的数据文件，并查找到应该进行写的位置。
	if d.writeFile == nil {
		// 根据当前的writeFileNum来创建新的数据文件
		curFileName := d.fileName(d.writeFileNum)
		d.writeFile, err = os.OpenFile(curFileName, os.O_RDWR|os.O_CREATE, 0600)
		if err != nil {
			return err
		}
		d.logf(INFO, "DISKQUEUE(%s): writeOne() opened %s", d.name, curFileName)

		if d.writePos > 0 {
			// 查找到应该进行写的位置
			_, err = d.writeFile.Seek(d.writePos, 0)
			if err != nil {
				d.writeFile.Close()
				d.writeFile = nil
				return err
			}
		}
	}
	// 首先获取4个字节的消息长度，并验证消息长度是否在合法的范围内。
	dataLen := int32(len(data))
	if dataLen < d.minMsgSize || dataLen > d.maxMsgSize {
		return fmt.Errorf("invalid message write size (%d) maxMsgSize=%d", dataLen, d.maxMsgSize)
	}

	// 清空writeBuf，以便进行写入
	d.writeBuf.Reset()
	// 在这里先把数据长度按照大段存储的方式写入writeBuf中
	err = binary.Write(&d.writeBuf, binary.BigEndian, dataLen)
	if err != nil {
		return err
	}
	// 在这里把data写入到writeBuf中
	_, err = d.writeBuf.Write(data)
	if err != nil {
		return err
	}
	// only write to the file once
	// 在这里才将writeBuf中的数据一次性写入到磁盘中。
	_, err = d.writeFile.Write(d.writeBuf.Bytes())
	if err != nil {
		d.writeFile.Close()
		d.writeFile = nil
		return err
	}
	// 这里的dataLen + 4是因为上面先写入了dataLen，由于dataLen是32位整型，4个byte，
	// 所以这里先加上属于数据长度的4个字节
	totalBytes := int64(4 + dataLen)
	d.writePos += totalBytes
	// diskQueue中的数据长度+1
	atomic.AddInt64(&d.depth, 1)

	// 如果当前 write index 超过了设定的文件的最大长度，则下次开始写新文件。
	if d.writePos > d.maxBytesPerFile {
		d.writeFileNum++
		d.writePos = 0

		// sync every time we start writing to a new file
		// 每次准备写新文件的时候都会进行sync操作。
		err = d.sync()
		if err != nil {
			d.logf(ERROR, "DISKQUEUE(%s) failed to sync - %s", d.name, err)
		}
		// 关闭文件
		if d.writeFile != nil {
			d.writeFile.Close()
			d.writeFile = nil
		}
	}
	return err
}
```

#### ioLoop

在创建diskqueue实例后，ioLoop作为一个goroutine会在后台循环执行，并处理读取、写入等请求。

```golang

// ioLoop provides the backend for exposing a go channel (via ReadChan())
// in support of multiple concurrent queue consumers
//
// it works by looping and branching based on whether or not the queue has data
// to read and blocking until data is either read or written over the appropriate
// go channels
//
// conveniently this also means that we're asynchronously reading from the filesystem
func (d *diskQueue) ioLoop() {
	var dataRead []byte // 读取出的数据
	var err error       // 错误
	var count int64     // 用来记录读取、写入了多少条消息
	var r chan []byte   // 用来暂存d.readChan
	// 创建一个sync操作的ticker
	syncTicker := time.NewTicker(d.syncTimeout)

	for {
		// dont sync all the time :)
		// 每读syncEvery次消息，会进行一次sync操作。
		if count == d.syncEvery {
			d.needSync = true
		}
		if d.needSync {
			// 将d.writeFile中的文件和meta信息刷写到磁盘。
			err = d.sync()
			if err != nil {
				d.logf(ERROR, "DISKQUEUE(%s) failed to sync - %s", d.name, err)
			}
			// 刷写过后count计数清零
			count = 0
		}
		// 这个说明有未读的数据
		if (d.readFileNum < d.writeFileNum) || (d.readPos < d.writePos) {
			// 当nextReadPos != readPos的时候，说明readChan中还有未处理的消息。不用readOnce
			// 当nextReadPos == readPos的时候，说明readChan中已经没有未处理的消息了，需要进行readOnce
			if d.nextReadPos == d.readPos {
				// 从磁盘中读取一条message
				dataRead, err = d.readOne()
				if err != nil {
					d.logf(ERROR, "DISKQUEUE(%s) reading at %d of %s - %s", d.name, d.readPos, d.fileName(d.readFileNum), err)
					d.handleReadError()
					continue
				}
			}
			//
			r = d.readChan
		} else {
			// 这里表明没有可以读取的数据
			r = nil
		}

		select {
		// the Go channel spec dictates that nil channel operations (read or write)
		// in a select are skipped, we set r to d.readChan only when there is data to read
		// 当有数据可以进行读取的时候
		case r <- dataRead:
			count++
			// moveForward sets needSync flag if a file is removed
			// 更新readPos, readFileNum, nextReadPos, nextReadFileNum的信息
			d.moveForward()
		// 接收到清空数据的请求
		case <-d.emptyChan:
			d.emptyResponseChan <- d.deleteAllFiles()
			count = 0
		// 当有数据写入的时候，并没有直接将数据直接写入磁盘，而是先写入writeChan，然后在这里再写入磁盘
		case dataWrite := <-d.writeChan:
			count++
			// 将数据写入磁盘中，并将操作结果写入writeResponseChan中。
			d.writeResponseChan <- d.writeOne(dataWrite)
		// 计时器响应，此时需要将metaData和w.writeFile进行持久化
		case <-syncTicker.C:
			// 如果没有读取和写入任何消息，则不作处理。
			if count == 0 {
				// avoid sync when there's no activity
				continue
			}
			d.needSync = true
		// 接受到退出命令
		case <-d.exitChan:
			goto exit
		}
	}

exit:
	d.logf(INFO, "DISKQUEUE(%s): closing ... ioLoop", d.name)
	syncTicker.Stop() // 停掉计时器
	d.exitSyncChan <- 1
}

```

#### reference

* disk queue source code : https://github.com/nsqio/go-diskqueue
