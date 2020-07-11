---
title: 第一章 MySQL体系结构与存储引擎
date: 2018-10-24 11:05:12
categories:
- 读书笔记
- 《MySQL技术内幕 InnoDB存储引擎》第二版
tags:
- MySQL
- InnoDB
description: 《MySQL技术内幕 InnoDB存储引擎》第二版，第一章 MySQL体系结构与存储引擎
keywords: 
- MySQL
- 存储引擎
- InnoDB
abstract: true
excerpt: true
---
## 词汇定义

* 数据库（database）：物理操作系统文件或其他形式文件类型的集合，即数据库是指保存数据的最原始文件，可能是硬盘里的文件或内存中的空间。是依靠某种数据模型组织起来并存放于二级存储器中的数据集合。
* 实例（instance）：介于客户端与数据库原始文件之间的后台进程与其共享内存，即数据库服务的服务端程序。用户对于数据库的任何操作，都是在实例上操作进行的。

所以，一般情况下，实例与数据库一一对应，在集群等情况下，多个实例可以共享同一个数据库。

<!--more-->

## 配置

启动实例时，程序会按顺序读取配置文件，新的配置变量会覆盖旧的。

变量查询方法：

```mysql
SHOW VARIABLES LIKE 'datadir';
```

## 体系结构

![MySQL架构图](/images/mysql-architechure.png)

MySQL包含以下部分：

* 连接池组件
* 管理服务和工具组件
* SQL接口组件
* 查询分析器组件
* 优化器组件
* 缓冲（Cache）组件
* 插件式存储引擎
* 物理文件

MySQL区别于其他数据库的一个重要特点是插件式的表存储引擎，MySQL的存储引擎是基于表的，不基于数据库。

## 存储引擎

可以根据MySQL预定义的存储引擎接口编写自定义的存储引擎。随着数据量的增加，性能会有所下降，但不是线性的。

### InnoDB存储引擎

主要面向在线事务处理（OLTP）的应用。支持事务、行锁、外键、非锁定读，即读操作不产生锁。

使用多版本并发控制（MVCC）获得高并发性。实现了MySQL的四种隔离级别，默认`REPEATABLE`级别。
使用`next-key locking`策略避免产生幻读（phantom）。
此外InnoDB存储引擎还提供了插入缓冲（insert buffer）、二次写入（double write）、自适应哈希索引（adaptive hash index）、预读（read ahead）等高性能和高可用功能。

对于表中数据的存储，采用了聚集（cluster）的方式，因此数据是按照主键的顺序进行存放的。如果没有显示的定义主键，会生成一个6字节的ROWID作为默认主键。

### MyISAM存储引擎

主要面向在线分析处理（OLAP）的应用。支持全文索引，不支持表锁、事务。

缓冲池只缓冲（cache）索引文件，而不是数据文件。

### InnoDB vs MyISAM

[官方存储引擎特点对比总结](https://dev.mysql.com/doc/refman/8.0/en/storage-engines.html)

|特点 | InnoDB  | MyISAM |
|---|---|---|
| 事务（提交，回滚等）  | 支持  | 不支持 |
| 锁级别  | 行  | 表 |
| B-Tree索引  | 支持  | 不支持  |
| T-Tree索引  | 不支持  | 不支持  |
| 哈希索引  | 不支持  | 不支持*  |
| 索引缓冲  | 支持  | 支持 |
| 外键  | 支持  |  不支持 |
| 存储限制  | 64TB | 256TB |
| MVCC（多版本并发控制）  | 支持 | 不支持 |
| 全文索引  | 支持（MySQL 5.6+）  |  支持 |
| 聚集索引（clustered index）  | 支持  | 不支持 |
| 数据缓冲  | 支持  | 不支持 |
| 地理空间数据类型  | 支持  |  支持 |
| 地理空间索引  | 支持（MySQL 5.7+）  |  支持 |

\* InnoDB在内部利用哈希索引来实现其自适应哈希索引功能。