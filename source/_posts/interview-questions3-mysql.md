---
title:      " 运维面试题系列--Mysql问题 "
date:       2021-03-03
banner_img: "https://gitee.com/like-ycy/images/raw/master/blog/header.jpg"
tags: [ 面试 ]
---

# Mysql面试问题



## 目录

- 理论问题

- SQL语句问题

- 思路分析题



## 理论问题

### 1、mysql主从同步原理

```
master开启bin-log功能，日志文件用于记录数据库的读写增删，
需要开启3个线程，master开启 IO线程，slave开启 IO线程 SQL线程，
Slave 通过IO线程连接master，并且请求某个bin-log，position之后的内容。
MASTER服务器收到slave IO线程发来的日志请求信息，io线程去将bin-log内容，position返回给slave IO线程。
slave服务器收到bin-log日志内容，将bin-log日志内容写入relay-log中继日志，创建一个master.info的文件，该文件记录了master ip 用户名 密码 master bin-log名称，bin-log position。
slave端开启SQL线程，实时监控relay-log日志内容是否有更新，解析文件中的SQL语句，在slave数据库中去执行。
```

**简化版**

```
master开启bin-log，slave的io线程连接master bin-log放入到relay-log中，slave的sql线程解析relay-log文件为SQL语句，在slave数据库中执行
```

### 2、mysql主从同步复制模式

```
• 异步复制( Asynchronous replication )
– 主库在执行完客户端提交的事务后会立即将结果返给客户端,并不关心从库是否已经接收并处理。

• 全同步复制( Fully synchronous replication )
– 当主库执行完一个事务,所有的从库都执行了该事务才返回给客户端。

• 半同步复制( Semisynchronous replication )
– 介于异步复制和全同步复制之间,主库在执行完客户端提交的事务后不是立刻返回给客户端,而是等待至少一个从库接收到并写到 relay log 中才返回给客户端。
```

### 3、mysql主从复制的方式

```
基于 SQL 语句的复制(statement-based replication, SBR)；
基于行的复制(row-based replication, RBR)；
混合模式复制(mixed-based replication, MBR)；
```

### 4、mysql索引分类

```
主键索引、唯一索引、普通索引、全文索引、组合索引
```

### 5、B+ Tree索引和Hash索引区别？

```
- 哈希索引适合等值查询，但是无法进行范围查询 
- 哈希索引没办法利用索引完成排序 
- 哈希索引不支持多列联合索引的最左匹配规则 
- 如果有大量重复键值的情况下，哈希索引的效率会很低，因为存在哈希碰撞问题
```

### 6、mysql两种引擎的区别

```
1、MyISAM是非事务安全的，而InnoDB是事务安全的
2、MyISAM锁的粒度是表级的，而InnoDB支持行级锁
3、MyISAM支持全文类型索引，而InnoDB不支持全文索引
4、MyISAM相对简单，效率上要优于InnoDB，小型应用可以考虑使用MyISAM
5、MyISAM表保存成文件形式，跨平台使用更加方便
```

### 7、两种引擎的应用场景

```
InnoDB：支持事务处理，支持外键，支持崩溃修复能力和并发控制。如果需要对事务的完整性要求比较高（比如银行），要求实现并发控制（比如售票），那选择InnoDB有很大的优势。如果需要频繁的更新、删除操作的数据库，也可以选择InnoDB，因为支持事务的提交（commit）和回滚（rollback）。 
MyISAM：插入数据快，空间和内存使用比较低。如果表主要是用于插入新记录和读出记录，那么选择MyISAM能实现处理高效率。如果应用的完整性、并发性要求比 较低，也可以使用。
```



## SQL语句问题

- 数据库根据学生表和分数表查学生的学号、姓名、课程，分数

```mysql
SELECT student.学号, student.姓名,score.课程, score.成绩 FROM student JOIN score ON student.学号=score.学号;
```
