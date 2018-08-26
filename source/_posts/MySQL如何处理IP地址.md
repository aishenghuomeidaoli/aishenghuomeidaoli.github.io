---
title: MySQL如何处理IP地址
date: 2018-08-23 08:29:52
categories:
- 技术
tags:
- MySQL
- CIDR
- IP
description: 本文讲述了MySQL存储、管理IP地址，以及依靠MySQL实现对连续IP的查询分组等操作。
keywords: 
- MySQL
- MySQL Transaction
- CIDR
- IP
- 事务
- IP分配
abstract: true
excerpt: true
---
## 背景介绍

公司是做云服务的，有一个需求是在业务层对物理IP进行管理，在管理系统中实现IP的录入与编辑，并为业务系统提供IP分配、释放的接口。
管理系统中的操作是基本数据的插入与修改，没有技术难点。在面向业务系统的IP分配接口中，当客户购买指定数量的IP时，需要从库里筛选出指定数量并符合CIDR规则的IP。
存在几个要考虑的因素。首先是分配规则的实现，

即将补充