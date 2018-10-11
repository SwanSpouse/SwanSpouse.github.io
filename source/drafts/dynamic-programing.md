---
title: 动态规划
date: 2018-10-11 22:17:25
tags:
  - algorithm
---

分治算法是指将问题划分成一些独立的子问题，递归地求解各子问题，然后合并子问题的解而得到原问题的解。与此不同，动态规划适用于子问题不是独立的情况，也就是各子问题包含公共的子问题。在这种情况下，若用分治算法则会做许多不必要的工作，即重复地求解公共的子子问题。动态规划算法对每个子子问题只求解一次，将其结果保存在一张表中，从而避免每次遇到各个子问题时重新计算答案。

动态规划通常应用于最优化问题。动态规划算法的设计可以分为如下四个步骤：
  1. 描述最优解的结构。
  2. 递归定义最优解的值。
  3. 按自底向上的方式计算最优解的值。
  4. 有计算出的结果构造一个最优解。 有时在第3步的计算中记录一些附加信息，可以第4步变得更加容易。

最优子结构可以用来判断是否可以用动态规划的方法来解决问题。

其实这个装配线的例题很好。很能说明上面这四个步骤。看看还有没有别的例题更加有说服力。

动态规划问题有个经典的文章叫做《背包九讲》，但是现在我已经找不到哪个是第一版本了。


### 120. Triangle

[120. Triangle](https://leetcode.com/problems/triangle/description/)

设状态为dp(i,j),表示从位置(i,j)出发，路径的最小和，则状态转移方程为:

dp(i, j) = min{ dp(i, j + 1), dp(i + 1, j + 1)} + (i, j)

### 53. Maximum Subarray

[53. Maximum Subarray](https://leetcode.com/problems/maximum-subarray/)

dp[j] = max{dp[j-1] + S[j], S[j]}, 1 <= j <= n

target = max{dp[j]}


### 152. Maximum Product Subarray

[152. Maximum Product Subarray](https://leetcode.com/problems/maximum-product-subarray/description/)

和上一题相比，除了需要记录当前的最大值之外，还需要对当前的最小值进行记录。

maxArr[j] = max{maxArr[j-1] * S[j], S[j], minArr[j-1] * S[j] }
minArr[j] = min{minArr[j-1] * S[j], S[j], maxArr[j-1] * S[j] }



### reference
* https://leetcode.com/
