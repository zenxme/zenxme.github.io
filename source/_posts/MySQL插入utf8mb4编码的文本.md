---
title: MySQL插入utf8mb4编码的文本
date: 2019-03-18 17:01:11
tags:
  - mysql
  - 编码
---

今天在爬取淘宝的评论的时候，在向MySQL数据库里插入带有表情的评论文本，结果出现了`Incorrect string value: ‘\xF0\x9F\x98\x83.....`的报错，在此记录下对这个问题的学习。

<!-- more -->

## 问题的原因

淘宝有些评论里面是有表情的，而这些表情是utf8文本，但是不是普通的utf8文本，而是特殊的utf8文本，它无法在MySQL的`utf8`编码下存储，只能存储到`utf8mb4`编码的字段中。

## MySQL里的utf8不是真正的utf8

具体可以参见这篇文章<https://medium.com/@adamhooper/in-mysql-never-use-utf8-use-utf8mb4-11761243e434>

总结就是

- MySQL里的`utfmb4`意思是真正的的`utf8`（就是完全实现）
- MySQL里的`utf8`不能编码许多Unicode字符

## 如何解决这个问题

在网上我查到了许多教程，大致就是用python向数据库里插入utf8mb4编码的数据的时候，要保证数据库、表、字段都得是utf8mb4编码的，以及在连接的开始的时候也要执行`SET NAMES utf8mb4`。

但是我经过实践以后可以发现，只要保证字段是utf8mb4编码的，就可以向里面插入utf8mb4编码的文本。可以和表、数据库的类型不一致。表、数据库我这边默认的编码是utf8，这两个在stack overflow上查了相关回答之后得知应该是兼容的。

创建一个表`test_table`

```sql
mysql> create table test_table(
    -> name varchar(200) null
    -> ) default charset utf8 collate utf8_general_ci;
Query OK, 0 rows affected (0.05 sec)
```

插入一段带表情的文本，这个时候出错了

```sql
mysql> insert into test_table(name) values ('😃 <');
ERROR 1366 (HY000): Incorrect string value: '\xF0\x9F\x98\x83 <' for column 'name' at row 1
```

这个时候看一下`SHOW CREATE test_table`

```sql
+------------+-----------------------------------------------------------------------------------------------------+
| Table      | Create Table                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------+
| test_table | CREATE TABLE `test_table` (
  `name` varchar(200) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8 |
+------------+-----------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)
```

修改下表的编码

```sql
mysql> alter table test_table modify column name varchar(200) character set utf8mb4 collate utf8mb4_general_ci;
Query OK, 0 rows affected (0.15 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

再次运行`SHOW CREATE test_table`

```sql
mysql> show create table test_table;
+------------+---------------------------------------------------------------------------------------------------------------------------+
| Table      | Create Table                                                                                                              |
+------------+---------------------------------------------------------------------------------------------------------------------------+
| test_table | CREATE TABLE `test_table` (
  `name` varchar(200) CHARACTER SET utf8mb4 DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8 |
+------------+---------------------------------------------------------------------------------------------------------------------------+
```

可以看到，表的编码是utf8，但是name字段的是utf8mb4。这个时候我再进行插入

```sql
mysql> insert into test_table(name) values ('😃 <');
ERROR 1366 (HY000): Incorrect string value: '\xF0\x9F\x98\x83 <' for column 'name' at row 1
```

还是不行。那么我运行一下`SET NAMES utf8;`

```sql
mysql> SET NAMES utf8mb4;
Query OK, 0 rows affected (0.00 sec)
```

再次插入

```sql
mysql> insert into test_table(name) values ('😃 <');
Query OK, 1 row affected (0.03 sec)
```

这个时候已经OK了，执行一下查询看看。

```sql
mysql> select * from test_table;
+--------+
| name   |
+--------+
| 😃 <     |
+--------+
1 row in set (0.00 sec)
```

## SET NAMES执行的时候发生了什么

设置了3个session变量。

```sql
SET character_set_client = charset_name;
SET character_set_results = charset_name;
SET character_set_connection = charset_name;
```

详细的可以查看这篇“阿里云RDS-数据库内核组”的文章<http://mysql.taobao.org/monthly/2015/05/07/>