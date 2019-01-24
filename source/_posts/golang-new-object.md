---
title: golang中的new和make
date: 2018-08-21 22:34:50
tags:
  - golang
---

#### 零值

要了解make和new首先要了解golang中关于零值的定义：当一个变量或者新值被创建时， 如果没有为其明确指定初始值，go语言会自动初始化其值为此类型对应的零值。各类型的零值如下：

* bool : false
* integer : 0
* float : 0.0
* string : ""
* pointer, function, interface, slice, channel, map: nil

对于复合类型，如数组，结构体，go语言会自动递归地将每一个元素初始化为其类型对应的零值。

#### new

new的结果是一个指针。在new(T)时会为对象T分配新的内存，将其置零，并返回它的地址（一个类型为*T的值）

如果用C语言来描述golang中new的过程，可以解释为

```C
T *t = (T*)malloc(sizeof(T))
memset(t, 0, sizeof(T))
```

#### make

一般来说我们只用make来创建slice, map 和 channel。 并且返回一个初始化的(而不是置零)，类型为T的值（而不是*T）

#### new和make的对比

从返回值上来说，new返回的是*T，是一个指针；make的返回值直接为T类型。

其次对于slice,map和channel类型的变量，我们一般都用make来创建，很少用new来进行创建。

用一段代码来说明make和new的区别：

```golang
var p = new([]int)
fmt.Printf("%+v\n", *p == nil)

*p = make([]int, 0)
fmt.Printf("%+v\n", *p == nil)

// 返回值为 true,  false
```

从上面的代码中就可以看出，new只是对slice分配了内存并将其置为零值（slice的零值为nil），所以 *p == nil 的判断为true； 而make则是在分配内存之后，对slice进行了初始化，不再为零值（nil）而是一个len=0的slice。


#### reference
* 《Go 并发编程实战》
* https://stackoverflow.com/questions/25358130/what-is-the-difference-between-new-and-make
* http://hustcat.github.io/new-and-make/
