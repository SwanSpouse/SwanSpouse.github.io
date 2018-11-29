---
title: NSQ中的优先级队列
tags:
  - NSQ
date: 2018-11-29 22:59:32
---

在NSQ中，有两个最小堆实现的优先级队列，分别为: PriorityQueue，inFlightPqueue。分别用作延迟消息和消息的at least once机制。

### 堆排序(以大顶堆为例)

堆排序的基本操作:

* MAX-HEAPIFY ： 运行时间为O(logn)，是保持最大堆性质的关键。
* BUILD-MAX-HEAP：以线性时间运行，可以在无序的输入数组基础上构造出大顶堆。
* HEAP-SORT：运行时间为O(nlogn)，对一个数组原地进行排序。
* MAX-HEAP-INSER，HEAP-INCREASE-KEY, 运行时间均为O(logn)，可以让堆结构作为优先队列使用。

#### MAX-HEAPIFY

堆调整是一个自顶向下的过程。

```golang

MAX-HEAPIFY(A, i)
  l = LEFT(i)
  r = RIGHT(I)
  if l <= heap-size[A] and A[l] > A[i]
    then largest = l
    else largest = i
  if r <= heap-size[A] AND A[r] > A[largest]
    then largest = r
  if largest != i
    then exchange A[i], A[largest]
      MAX-HEAPIFY(A, largest)
```

#### BUILD-MAX-HEAP

建立一个堆，是一个自底向上的过程。

```golang

BUILD-MAX-HEAP
  heap-size[A] = length[A]
  for i = length[A] / 2 downto 1
    do MAX-HEAPIFY(A, i)

```

#### HEAP-SORT

最开始利用BUILD-MAX-HEAP将输入数组构造成一个大顶堆。每次将根节点与A[n]互换来达到最终正确的位置。同时缩小堆的大小。
新的根元素可能违背了最大堆的性质，这时通过MAX-HEAPIFY来进行调整，保持这一性质。不断重复此过程，直到堆的大小降到2。

```golang

HEAP-SORT(A)
  BUILD-MAX-HEAP(A)
  for i = length[A] downto 2
    do exchange A[1], A[i]
      heap-size[A] = heap-size[A] - 1
      MAX-HEAPIFY(A, 1)

```

#### HEAP-INCREASE-KEY

首先将i位置的元素更新为key。由于A[i]的增大可能会违反最大堆的性质。所以在过程中采用了向上调整的方法。

```golang

HEAP-INCREASE-KEY (A, i, key)
  if key < A[i]
    then error "new key is smaller than curent key"
  A[i] = key
  while i > 1 and A[PARENT(i)] < A[i]
    do exchange A[i], A[PARENT(i)]
      i = PARENT(i)

```

#### MAX-HEAP-INSERT

向堆中插入元素。

```golang
MAX-HEAP-INSERT(A, key)
  heap-size[A] = heap-size[A]+1
  A[heap-size[A]] = -∞
  HEAP-INCREASE-KEY(A, heap-size[A], key)
```


### InFlightPqueue源码解析

可以将下面源码和上面堆排序的基本操作结合起来看。

``` golang

type inFlightPqueue []*Message

func newInFlightPqueue(capacity int) inFlightPqueue {
	return make(inFlightPqueue, 0, capacity)
}

// 交换两个元素的位置
func (pq inFlightPqueue) Swap(i, j int) {
	pq[i], pq[j] = pq[j], pq[i]
	pq[i].index = i
	pq[j].index = j
}

// 向Queue中添加元素
func (pq *inFlightPqueue) Push(x *Message) {
	n := len(*pq)
	c := cap(*pq)
	// 当前容量+1超过cap的时候，会进行扩容，
	// 这里扩容扩的是cap，不是length
	if n+1 > c {
		npq := make(inFlightPqueue, n, c*2)
		copy(npq, *pq)
		*pq = npq
	}
	// 把length的指针向右移动一位，扩充一个空白元素用来填充x
	*pq = (*pq)[0 : n+1]
	x.index = n
	// 把x放入到队列的末尾
	(*pq)[n] = x
	// 进行一次堆调整
	pq.up(n)
}

// 弹出堆顶元素
func (pq *inFlightPqueue) Pop() *Message {
	n := len(*pq)
	c := cap(*pq)
	// 将堆顶元素和末端元素进行交换
	pq.Swap(0, n-1)
	// 进行一次向下调整
	pq.down(0, n-1)
	// 当符合条件时进行缩容
	if n < (c/2) && c > 25 {
		npq := make(inFlightPqueue, n, c/2)
		copy(npq, *pq)
		*pq = npq
	}
	// 获取末尾元素，并删除末尾元素
	x := (*pq)[n-1]
	x.index = -1
	*pq = (*pq)[0 : n-1]
	return x
}

// 移除i位置的节点
func (pq *inFlightPqueue) Remove(i int) *Message {
	n := len(*pq)
	// 若要移除的元素是最后一个，则直接移除就好了
	if n-1 != i {
		// 把最后一个元素和要移除的元素进行交换
		pq.Swap(i, n-1)
		// 进行一次向下堆调整
		pq.down(i, n-1)
		// 进行一次向上堆调整
		pq.up(i)
	}
	// 移除最后一个元素，将移除的元素返回
	x := (*pq)[n-1]
	x.index = -1
	*pq = (*pq)[0 : n-1]
	return x
}

// 如果堆顶元素大于max，则返回nil和堆顶元素与max之间的差值
// 如果堆顶元素小于max，则返回堆顶元素，并将堆顶元素删除。
func (pq *inFlightPqueue) PeekAndShift(max int64) (*Message, int64) {
	if len(*pq) == 0 {
		return nil, 0
	}
	// 取出堆顶元素
	x := (*pq)[0]
	// 若堆顶元素大于max，则return nil，并返回和max的差值
	if x.pri > max {
		return nil, x.pri - max
	}
	// 移除堆顶元素
	pq.Pop()
	// 将x返回
	return x, 0
}

// 进行一次向上堆调整
func (pq *inFlightPqueue) up(j int) {
	for {
		// 获取父节点的位置
		i := (j - 1) / 2 // parent
		if i == j || (*pq)[j].pri >= (*pq)[i].pri {
			break
		}
		// 若j位置节点的值小于父节点的值，则进行交换
		pq.Swap(i, j)
		// 继续调整
		j = i
	}
}

// 进行一次向下堆调整
func (pq *inFlightPqueue) down(i, n int) {
	for {
		// 找到i的左孩子
		j1 := 2*i + 1
		// 若超过堆的大小或溢出，则直接退出
		if j1 >= n || j1 < 0 { // j1 < 0 after int overflow
			break
		}
		// 把当前节点和自己的两个孩子作比较，若自己的值大于任意孩子的值，把自己和值较小的孩子交换位置。
		j := j1 // left child
		if j2 := j1 + 1; j2 < n && (*pq)[j1].pri >= (*pq)[j2].pri {
			j = j2 // = 2*i + 2  // right child
		}
		if (*pq)[j].pri >= (*pq)[i].pri {
			break
		}
		pq.Swap(i, j)
		i = j
	}
}

```

#### reference

* inFlightPqueue https://github.com/nsqio/nsq/blob/master/nsqd/in_flight_pqueue.go
* pqueue https://github.com/nsqio/nsq/tree/master/internal/pqueue/pqueue.go
