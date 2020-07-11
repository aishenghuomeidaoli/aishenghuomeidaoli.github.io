---
title: Nginx基础知识
date: 2018-10-18 09:39:51
categories:
- 技术
- Nginx
tags:
- Nginx
---

## 什么是Nginx

Nginx是一个web服务器和反向代理服务器，用于HTTP、HTTPS、SMTP、POP3和IMAP协议。

<!--more-->

## 列举Nginx的一些特性

* 更快

单次请求会得到更快的响应；在高峰期，Nginx可以比其他Web服务器更快地响应请求。

* 高扩展性



* 反向代理/L7负载均衡器
* 嵌入式Perl解释器
* 动态二进制升级
* 可用于重新编写URL，具有非常好的PCRE支持

## 列举Nginx和Apache之间的不同点

|Nginx     |Apache     |
|--------- |---------  |
|基于事件的服务器|基于流程的服务器|
|一个线程处理所有请求（受内存限制）|单个线程处理单个请求（受CPU限制）|

## 请解释Nginx服务器上的Master和Worker进程分别是什么?

master对work进程采用信号进行控制

Master进程：读取及评估配置和维持

Worker进程：处理请求







