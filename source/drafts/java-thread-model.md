---
title: Java 线程模型
date: 2018-09-05 23:46:15
tags:
  - java
  - thread
---

Java线程模型的实现

1. 使用内核线程实现 轻量级进程
2. 使用用户线程实现
3. 用户线程 + 轻量级进程实现

Java线程调度

协作式和抢占式

Java线程的状态转换

Java进程的5种状态

New, Runnable, Waiting (Waiting, Timed Waiting) Blocked  Terminated

有两种Waiting

Blocked和Waiting区别
