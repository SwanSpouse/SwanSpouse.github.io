---
title: mysql 常用命令
tags:
  - mysql
---


### 概述

#### 查看mysql 使用的配置文件
mysql --help | grep my.cnf

按照 /etc/my.cnf -> /etc/mysql/my.cnf -> /usr/local/mysql/etc/my.cnf -> ~/.my.cnf 来获取配置文件的，并以最后获得的配置文件为准。

#### 数据所在路径

show variables like 'datadir'\G;

datadir Linux 系统上默认为：/usr/local/mysql/data， data目录只是一个连接，指向了/opt/mysql_data目录。

用需要保证/opt/mysql_data的用户和权限。

#### InnoDB引擎简介

InnoDB存储引擎支持事务，其设计目标主要面向在线事务处理(OLTP)的应用。其特点是行锁设计、支持外键，并支持类似于Oracle的非锁定读，即默认读取操作不会产生锁。5.5.8版本开始，InnoDB存储引擎是默认的存储引擎。

InnoDB通过使用多版本并发控制（MVCC）来获得高并发性，并且实现了SQL标准的4中隔离级别。默认为REPEATABLE级别，同时使用next-key locking策略来避免幻读（phantom)现象的产生。

InnoDB存储引擎采用了聚集（clustered）的方式，因此每张表的存储都是按主键的顺序进行存放。

#### checkpoint技术

checkpoint技术的目的是解决以下几个问题：

* 缩短数据库的恢复时间
* 缓冲池不够用时，将脏页刷新到磁盘
* 重做日志不可用时，刷新脏页


#### B+树索引

数据库中的B+树索引可以分为聚集索引(clustered index)和辅助索引(secondary index)。

聚集索引： 就是按照每张表的主键构建一棵B+树，同时叶子节点中存放的即为整张表的行为记录数据，也将聚集索引的叶子节点成为数据页。


#### reference
* 《MYSQL 技术内幕 InnoDB 存储引擎》
