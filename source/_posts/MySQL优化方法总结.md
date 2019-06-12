---
title: MySQL优化方法总结
date: 2019-03-30 14:42:16
tags:
  - mysql
  - 数据库
---

对MySQL的各种优化方法的总结。

<!-- more -->

## 如何找出运行速度慢的语句

- 开启慢查询日志（Slow Query Log），找出查询效率低的语句。（通过改变long_query_time的值来控制时间，超出这个值的语句就会被记录）

## 分析查询语句

```sql
EXPLAIN [EXTENDED] SELECT select_options
```

`EXTENDED`是可选的，加上之后将产生更多的信息。`select_options`代表剩下的SELECT语句的查询选项。

也可以使用`DESCRIBE`，或者也可以使用它的简写`DESC`

```sql
DESCRIBE SELECT select_options
```

## 索引

建立索引可以显著地提高查询的速度，但并不是所有的查询情况下索引都会起作用，下面是一些特殊情况。

1. 使用`LIKE`关键字的查询语句，如果第一个字符为`%`，索引不会起作用。
2. 使用多列索引的查询语句，如果查询条件不包含第一个字段，索引不会起作用。
3. 查询语句的条件只有`OR`关键字，并且`OR`前后的两个条件中的列都是索引时，查询中才使用索引。否则，查询将不使用索引。

## 优化子查询

子查询虽然很灵活，但是效率不高，MySQL需要为内层语句的查询结果建立一个临时表，查询完毕后再撤销这些临时表。

可以使用连接（JOIN）来代替子查询，这样就省去了建立和撤销临时表的开销，加快了速度。

## 优化数据库结构

1. 对字段较多的表，可以把其中使用频率低的一部分字段分离出来建立一个新的表。
2. 对经常需要联合查询的表，可以建立中间表，把需要经常联合查询的数据插入到中间表中，然后把原来的联合查询改为对中间表的查询。
3. 适当地增加冗余字段。(但是可能会产生数据不一致问题)

## 优化插入

### 对MyISAM表

插入大量数据前：

```sql
ALTER TABLE table_name DISABLE KEYS;  /* 禁用索引 */
SET UNIQUE_CHECKS=0;                  /* 禁用唯一性校验 */
```

插入时：

1. 使用一条INSERT语句插入多条记录。
2. 如果需要导入数据，使用LOAD DATA INFILE语句，它比INSERT语句快。

之后：

```sql
ALTER TABLE table_name ENABLE KEYS;  /* 开启索引 */
SET UNIQUE_CHECKS=1;                 /* 启用唯一性校验 */
```

### 对InnoDB表

插入大量数据前：

```sql
SET UNIQUE_CHECKS=0;                  /* 禁用唯一性校验 */
SET FOREIGN_KEY_CHECKS=0;             /* 禁用外键检查 */
SET AUTOCOMMIT=0;                     /* 禁止自动提交 */
```

插入时的方法和MyISAM的一样。

之后：

```sql
SET UNIQUE_CHECKS=1;                 /* 启用唯一性校验 */
SET FOREIGN_KEY_CHECKS=1;             /* 恢复外键检查 */
SET AUTOCOMMIT=1;                     /* 恢复自动提交 */
```

## 优化表

1. 使用`ANALYZE TABLE`语句分析表
2. 使用`CHECK TABLE`语句检查表
3. 使用`OPTIMIZE TABLE`语句优化表

这3种语句执行的时候都会给表加上只读锁，具体的用法可以查阅相关文档。

## 优化服务器硬件配置

1. 配备大内存
2. 配备高速磁盘系统
3. 合理分布磁盘I/O
4. 配备高速多核处理器
5. 配备大带宽网络

## 优化MySQL配置

参见这篇文章：<http://www.speedemy.com/17-key-mysql-config-file-settings-mysql-5-7-proof/>