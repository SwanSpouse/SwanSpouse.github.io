---
title: 常见位运算
tags:
  - algorithm
---

### 位运算简介（WIKI）

Bit manipulation is the act of algorithmically manipulating bits or other pieces of data shorter than a word. 需要进行位运算的程序包括：低层设备控制、错误检测、校正算法、数据压缩、加密算法和一些优化。 For most other tasks, modern programming languages allow the programmer to work directly with abstractions instead of bits that represent those abstractions. Source code that does bit manipulation makes use of the bitwise operations: AND, OR, XOR, NOT, and bit shifts.

在某些情况下，位操作可以消除或减少在数据结构上使用的循环，同时因为位操作是并行处理的，所以使用位运算可以成倍的提高代码运行的速度。但是，使用位运算可能会导致代码变得更加难以编写和维护。

### 位运算基本操作

1. Set union A | B
2. Set intersection A & B
3. Set subtraction A & ~B
4. Set negation ALL_BITS ^ A or ~A
5. Set bit A |= 1 << bit
6. Clear bit A &= ~(1 << bit)
7. Test bit (A & 1 << bit) != 0
8. Extract last bit A&-A or A&~(A-1) or x^(x&(x-1))
9. Remove last bit A&(A-1)
10. Get all 1-bits ~0

### Bit操作实例

#### Count the number of ones in the binary representation of the given number


```c
int count_one(int n) {
    while(n) {
        n = n & (n-1);
        count++;
    }
    return count;
}
```

n-1意思是：从后向前，找到最后一个不为0的bit，把它置为0；从后向前，把末尾所有的0都置为1。

n & (n -1) 可以理解为移除从后向前的第一个1。

![count one](https://s1.ax1x.com/2018/11/04/i5c3gH.png)


#### Is power of four

```c
bool isPowerOfFour(int n) {
    return (n & (n - 1)) == 0 && (n & 0x55555555) != 0;
}
```

首先，为4的幂次的数有一个特点就是只有一位是1，且这个1要出现在奇数位上，例如：1: 1, 4: 100, 16: 10000。所以首先要判断1的个数是否是1个，然后再判断1的位置是否在奇数位上。

n & (n - 1) == 0 表示n的二进制表示中，只有一位是1，其余位数全为0。

n & 0x55555555 表示保留所有奇数位的数值，并将偶数位的数值置为0，因为5的二进制表示是0101。 (n & 0x55555555) != 0 表示奇数位上并不都是0。

n & (n - 1) == 0 && (n & 0x55555555) 的意思是：二进制表示的数中只有一位是1，并且这个1在奇数位上。

#### Sum of Two Integers

```c
int getSum(int a, int b) {
    return b==0? a:getSum(a^b, (a&b)<<1); //be careful about the terminating condition;
}
```

和笔算加法相同。对于二进制数的而言，对应位相加就可以使用异或（xor）操作，计算进位就可以使用与（and）操作，在下一步进行对应位相加前，对进位数使用移位操作（<<）。

#### Missing Number

Given an array containing n distinct numbers taken from 0, 1, 2, ..., n, find the one that is missing from the array. For example, Given nums = [0, 1, 3] return 2. (Of course, you can do this by math.)

```c
int missingNumber(vector<int>& nums) {
    int ret = 0;
    for(int i = 0; i < nums.size(); ++i) {
        ret ^= i;
        ret ^= nums[i];
    }
    return ret^=nums.size();
}
```

利用了A xor A = 0 这个性质。













### reference

* https://leetcode.com/problems/sum-of-two-integers/discuss/84278/A-summary%3A-how-to-use-bit-manipulation-to-solve-problems-easily-and-efficiently
