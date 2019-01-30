---
title: Mysql 索引
date: 2019-01-30 11:03:41
tags:
    - mysql
---

### MySql索引

mysql的索引分为单列索引(主键索引,唯一索引,普通索引)和组合索引.

* 单列索引:一个索引只包含一个列,一个表可以有多个单列索引.

* 组合索引:一个组合索引包含两个或两个以上的列,

#### 联合索引:

select * from users where area=’Beijing’ and age=22;

* 如果我们是在area和age上分别创建单个索引的话，由于mysql查询每次只能使用一个索引，所以虽然这样已经相对不做索引时全表扫描提高了很多效率。

* 如果我们创建了(area, age, salary)的复合索引，那么其实相当于创建了(area,age,salary)、(area,age)、(area)三个索引，这被称为最左前缀特性。
因此我们在创建复合索引时应该将最常用作限制条件的列放在最左边，依次递减。

* 只要列中包含有NULL值都将不会被包含在索引中，复合索引中只要有一列含有NULL值，那么这一列对于此复合索引就是无效的。所以我们在数据库设计时不要让字段的默认值为NULL。

* 对串列进行索引，如果可能应该指定一个前缀长度。例如，如果有一个CHAR(255)的 列，如果在前10 个或20 个字符内，多数值是惟一的，那么就不要对整个列进行索引。短索引不仅可以提高查询速度而且可以节省磁盘空间和I/O操作。

* mysql查询只使用一个索引，因此如果where子句中已经使用了索引的话，那么order by中的列是不会使用索引的。因此数据库默认排序可以符合要求的情况下不要使用排序操作；尽量不要包含多个列的排序，如果需要最好给这些列创建复合索引。

* 一般情况下不鼓励使用like操作，如果非使用不可，如何使用也是一个问题。like “%aaa%” 不会使用索引而like “aaa%”可以使用索引。

* 在列上进行运算将导致查询不使用索引。

#### MySQL在字符串类型字段上搜索整型值时无法使用索引



假设表结构如下

```SQL
CREATE TABLE `user` (
  `user_id` int(11) NOT NULL AUTO_INCREMENT,
  `user_name` varchar(64) NOT NULL DEFAULT '',
  PRIMARY KEY (`user_id`),
  KEY `user_name` (`user_name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8
```

当执行下面这样的sql时无法使用索引

```
select * from user where user_name = 100;
```

因为上面的sql语句等价于

```
select * from user where convert(user_name, signed) = 100;
```

即搜索把user_name字段转化为整型后的值等于100的行。而这样的user_name是可以有很多的，例如”100”，”0100”和”00100”。所以不能使用索引进行查找！


The essential point is that the index cannot be used if the database has to do a conversion on the table-side of the comparison.

Besides that, the DB always coverts Strings -> Numbers because this is the deterministic way (otherwise 1 could be converted to '01', '001' as mentioned in the comments).

So, if we compare the two cases that seem to confuse you:

```sql
-- index is used
EXPLAIN SELECT * FROM a_table WHERE int_column = '1';
```
The DB converts the string '1' to the number 1 and then executes the query. It finally has int on both sides so it can use the index.

```sql
-- index is NOT used. WTF?
EXPLAIN SELECT * FROM a_table WHERE str_column = 1;
```
Again, it converts the string to numbers. However, this time it has to convert the data stored in the table. In fact, you are performing a search like cast(str_column as int) = 1. That means, you are not searching on the indexed data anymore, the DB cannot use the index.


#### reference

* https://stackoverflow.com/questions/16786063/mysql-comparison-of-integer-value-and-string-field-with-index
