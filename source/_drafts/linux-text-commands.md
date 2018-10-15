---
title: linux 文本处理相关命令
date: 2018-08-27 10:15:21
tags:
  - linux
---

### linux文本处理相关命令

linux常见的文本处理命令有sed、awk、grep、cut这些。借助正则表达式和这些命令可以实现对文本的过滤、剥离、替换、搜索等操作。

#### 正则表达式

#### sed

sed是流编辑器(stream editor)的缩写。sed命令一般用来进行文本替换。

#### awk

#### grep

#### cut

cut是一个将文本按列进行切分的小工具。

常见用法: cut -f 2,3 --complement -d ";" student.txt
* -f 指定要显示字段。格式: N1,N2,N3 |  N- | -M | N-M
* --complement 选项对提取的字段进行补集运算。上述样例中取非2、3的字段。
* -d 指定列分隔符。

### reference

* 《linux shell脚本攻略 第二版》
