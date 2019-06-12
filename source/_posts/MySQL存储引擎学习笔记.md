---
title: MySQL存储引擎总结
date: 2019-03-12 17:32:51
tags:
  - mysql
  - 数据库
---

对MySQL存储引擎InnoDB、MyISAM、MEMORY的学习和记录。

<!-- more -->

在MySQL中，使用`SHOW ENGINES`命令可以查看所支持的引擎，我当前的版本是5.7，可以看到有如下的输出。

```
mysql> SHOW ENGINES;
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
| Engine             | Support | Comment                                                        | Transactions | XA   | Savepoints |
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
| InnoDB             | DEFAULT | Supports transactions, row-level locking, and foreign keys     | YES          | YES  | YES        |
| MRG_MYISAM         | YES     | Collection of identical MyISAM tables                          | NO           | NO   | NO         |
| MEMORY             | YES     | Hash based, stored in memory, useful for temporary tables      | NO           | NO   | NO         |
| BLACKHOLE          | YES     | /dev/null storage engine (anything you write to it disappears) | NO           | NO   | NO         |
| MyISAM             | YES     | MyISAM storage engine                                          | NO           | NO   | NO         |
| CSV                | YES     | CSV storage engine                                             | NO           | NO   | NO         |
| ARCHIVE            | YES     | Archive storage engine                                         | NO           | NO   | NO         |
| PERFORMANCE_SCHEMA | YES     | Performance Schema                                             | NO           | NO   | NO         |
| FEDERATED          | NO      | Federated MySQL storage engine                                 | NULL         | NULL | NULL       |
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
```

## InnoDB

InnoDB是MySQL5.5.5之后的默认存储引擎，主要特性有：

- 支持事务，每一条SQL语句都默认封装成事务，具有提交、回滚和崩溃恢复能力。
- 事务具有ACID特性，还实现了4个标准的隔离级别：未提交读(Read uncommitted)，已提交读(Read committed)，可重复读(Repeatable read)，可串行化(Serializable)。
- 支持外键完整性约束(FOREIGN KEY)。
- 索引是聚集索引，数据文件是和索引绑在一起的(ibdata文件)，必须要有主键(如果没有定义主键，那么该表的第一个唯一非空索引被作为聚集索引。如果这两种情况都没有，那么会自动生成一个隐藏的主键作为聚集索引，这个隐藏的主键是一个6个字节的列，该列的值会随着数据的插入自增)。
- 不保存表的具体行数，执行`select count(*) from table_name`时需要全表扫描。
- 支持行级锁和表级锁。
- 支持在线热备份。

## MyISAM

MyISAM是MySQL5.5.5之前的默认存储引擎，主要特性有：

- 不支持事务。
- 不支持外键。
- 数据和索引分开存储的，分为3个文件，文件名是表名，扩展名指出了类型。`.frm`(format的缩写)文件存储表定义，`.MYD`(MYData)存储表数据，`.MYI`(MYIndex)存储索引文件。
- 用一个变量保存了整个表的行数，执行`select count(*) from table_name`时只需要读出该变量即可，速度很快。
- 不支持行级锁，只支持表级锁。
- 每个字符列可以有不同的字符集。

注意：在SQL查询中，可以自由地将InnoDB类型的表和其它类型的表混合起来，甚至在同一个查询中也可以混合。但是如果把一个含有外键的InnoDB类型的表转换为MyISAM的话则会失败。

## MEMORY

MEMORY存储引擎将表中的数据存储到内存中，为查询和其它表数据提供快速访问，主要特性有：

- 每个表有多达32个索引，每个索引16列，以及500B的最大键长度。
- 执行HASH和BTREE索引。
- 使用一个固定的记录长度的格式。
- 不支持BLOB或TEXT列。
- 在所有客户端之间共享，就像其它任何非TEMPORARY表。
- 内容被存在内存中，当不再需要表里的内容时，要释放该表所使用的内存，应该清空表里的数据，或者直接删除表。

## 如何选择存储引擎

以下摘自<<MySQL5.7从入门到精通>>

> 如果要提供提交、回滚和崩溃恢复能力的事务安全（ACID兼容）能力，并要求实现并发控制，InnoDB是个很好的选择。如果数据主要用来插入和查询记录，则MyISAM引擎能提供较高的处理效率；如果只是临时存放数据，数据量不大，并且不需要较高的数据安全性，可以选择将数据保存在内存中的Memory引擎，MySQL中使用该引擎作为临时表，存放查询的中间结果。如果只有INSERT和SELECT操作，可以选择Archive引擎，Archive存储引擎支持高并发的插入操作，但是本身并不是事务安全的。Archive存储引擎非常适合存储归档数据，如记录日志信息可以使用Archive引擎。
>
> 使用哪一种引擎要根据需要灵活选择，一个数据库中多个表可以使用不同引擎以满足各种性能和实际需求。使用合适的存储引擎，将会提高整个数据库的性能。