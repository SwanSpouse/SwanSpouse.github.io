---
title: apache-arrow
tags:
  - big data
---

Apache Arrow是Apache基金会下一个全新的开源项目，同时也是顶级项目。它的目的是作为一个跨平台的数据层来加快大数据分析项目的运行速度。

Apache Arrow做为今后的标准数据交换格式，各个数据分析的系统和应用之间的交互性可以说是上了一个新的台阶。过去大部分的CPU周期都花在了数据的序列化和反序列化上，现在我们则能够实现不同系统之间数据的无缝共享。这意味着用户在将不同的系统结合使用时再也不用为数据格式多花心思了。

下图是列式存储格式和行式存储格式之间的区别：

![行存储列存储之间的对比](https://sf6-ttcdn-tos.pstatp.com/obj/temai/FnQieP4CGaguX8pXXpaTJxeMnojGwww798-572)

使用 Apache Arrow前后的对比
* 每个系统都有自己内部的内存格式； ---> 所有系统使用统一的内存格式
* 70-80%的CPU浪费在序列化和反序列化过程； ---> 提高了CPU的处理效率
* 类似功能在多个项目中实现，没有一个标准。 ---> 项目间可以共享功能。

#### TiDB-Chunk

* https://www.pingcap.com/blog-cn/tidb-source-code-reading-10/


#### reference
* https://www.iteblog.com/archives/1588.html
