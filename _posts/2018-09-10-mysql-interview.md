---
title: Mysql常见面试题
categories:
 - 源码分析 
tags:
 - mysql源码分析
---

[toc]

### MySQL的复制原理以及流程
1. 主：binlog线程——记录下所有改变了数据库数据的语句，放进master上的binlog中； 
2. 从：io线程——在使用start slave 之后，负责从master上拉取 binlog 内容，放进 自己的relay log中； 
3. 从：sql执行线程——执行relay log中的语句；


### InnoDB和MYISAM引擎区别

- InnoDB支持事务
- InnoDB支持行锁 MYISAM支持表锁
- InnoDB使用聚集索引 MYISAM使用非聚集索引
- InnoDB索引和数据存储在一个文件，MYISAM分开成索引文件和数据文件
- InnoDB 5.6之后才支持全文索引 MYISAM一直支持全文索引
- InnoDB支持MVCC MYISAM不支持
- InnoDB支持外键，MYISAM不支持

### InnoDB 4大特性

### InnoDB参数优化

### InnoDB行锁具体是怎么实现的

### varchar 和char区别

- varchar存储可变长字符串，有1或2个字节来保存长度，长度小于等于255，用1个字节，否则使用2个字节。Update使得空间增长，MyISAM会将行拆成不同的片段存储，InnoDB则需要分裂页使行可以放到页内。
- char是定长得到，根据指定的长度分配足够的空间。当字符串不够时候系统会在末尾填充空格以方便比较（也会导致末尾的空格在查询出来时剪掉了）。

varchar(5)和varchar(200)的空间开销是一样的，但是更长的列会消耗更多的内存，mysql会分配固定大小的内存块保存内部值。

### 描述mysql中事务和日志及区别

- 几种日志

1. bin log 日志
2. redo log 写入物理磁盘的日志
3. undo log 回滚日志

- 事务的4种隔离级别

1. RU Read Uncommitted 读未提交
2. RC Read Committed 读已提交
3. RR Repeatable Read 可重复读
4. Serialize 串行化

- 事务如何通过日志来实现

### binlog的几种日志录入格式和区别

### sql 优化
1. explain 各个字段的意思

```
mysql> explain select * from course where id in (1024, 1025);
+----+-------------+--------+-------+---------------+---------+---------+------+------+-------------+
| id | select_type | table  | type  | possible_keys | key     | key_len | ref  | rows | Extra       |
+----+-------------+--------+-------+---------------+---------+---------+------+------+-------------+
|  1 | SIMPLE      | course | range | PRIMARY       | PRIMARY | 4       | NULL |    2 | Using where |
+----+-------------+--------+-------+---------------+---------+---------+------+------+-------------+


```

2. profile的意义以及使用场景

### 备份计划 mysqldump






