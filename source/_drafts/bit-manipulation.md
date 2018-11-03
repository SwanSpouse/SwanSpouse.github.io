---
title: 常见位运算
tags:
  - algorithm
---

### 位运算简介（WIKI）

Bit manipulation is the act of algorithmically manipulating bits or other pieces of data shorter than a word. 需要进行位运算的程序包括：低层设备控制、错误检测、校正算法、数据压缩、加密算法和一些优化。 For most other tasks, modern programming languages allow the programmer to work directly with abstractions instead of bits that represent those abstractions. Source code that does bit manipulation makes use of the bitwise operations: AND, OR, XOR, NOT, and bit shifts.

在某些情况下，位操作可以消除或减少在数据结构上使用的循环，同时因为位操作是并行处理的，所以使用位运算可以成倍的提高代码运行的速度。但是，使用位运算可能会导致代码变得更加难以编写和维护。

### 位运算基础

位运算的核心是  & (and与), | (or或), ~ (not非) and ^ (exclusive-or, xor异或) and shift operators (位移操作) a << b and a >> b.

There is no boolean operator counterpart to bitwise exclusive-or, but there is a simple explanation. The exclusive-or operation takes two inputs and returns a 1 if either one or the other of the inputs is a 1, but not if both are. That is, if both inputs are 1 or both inputs are 0, it returns 0. Bitwise exclusive-or, with the operator of a caret, ^, performs the exclusive-or operation on each pair of bits. Exclusive-or is commonly abbreviated XOR.

按位异或没有布尔运算符对应，但有一个简单的解释。 异或运算需要两个输入，如果其中一个或另一个输入为1，则返回1，但如果两者都不是，则返回1。 也就是说，如果两个输入都是1或两个输入都是0，则它返回0.按位异或 - 与插入符号的运算符^，对每对位执行异或操作。 独占或通常缩写为XOR。

没有布尔运算符对位异或，但有一个简单的解释。异或运算取两个输入，如果输入之一或另一个输入为1，则返回1，但如果两者都是，则不返回。也就是说，如果两个输入都是1，或者两个输入都是0，则返回0。Bitwise异或，用插入符号的操作符^，对每一对位执行异或运算。异或通常是缩写的异或。

### Bit操作实例


### reference

* https://leetcode.com/problems/sum-of-two-integers/discuss/84278/A-summary%3A-how-to-use-bit-manipulation-to-solve-problems-easily-and-efficiently
