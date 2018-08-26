---
title: 使用Docker Compose快捷部署WEB服务、分布式任务队列项目实践
date: 2018-08-15 12:57:27
categories:
- 技术
tags:
- Docker
- Docker Compose
- HaProxy
- Django
- Celery
- MySQL
- Redis
description: 使用Docker Compose快捷部署HaProxy+Django+Celery+MySQL+Redis项目实践
keywords: 
- Docker
- Docker Compose
- HaProxy
- Django
- Celery
- MySQL
- Redis
- 容器
- 负载均衡
- 分布式任务队列
abstract: true
excerpt: true
---

项目地址：[GItHub](https://github.com/aishenghuomeidaoli/docker-django-mysql-celery)

## 介绍

`Docker` 容器技术解决了项目部署时开发、测试、预演、生产环境差异的影响。
可以为项目的不同服务创建独立的容器，使项目模块化，降低了耦合性。
当同一宿主机中存在多个容器时，容器的统一管理便显得很必要。
`Docker Compose`具有管理多个容器的功能，`Docker Compose`通过调用Docker的接口，从 *yaml* 文件中读取容器配置，对容器进行统一管理。

项目架构 `HaProxy` + `Django` + `Celery` + `MySQL` + `Redis`：

* `HaProxy`，作为项目的代理程序，将请求转发至web服务器，并可以实现负载均衡；

* `Django`，提供web服务，在接口中调用`Celery`创建任务，作为生产者，在任务函数中执行任务，作为消费者；

* `Celery`，提供分布式任务队列服务；

* `MySQL`，为web服务提供数据库服务，存储`Celery`的处理结果；

* `Redis`，作为`Celery`的消息中间件，也可作为web的缓存服务器。

<!--more-->

## 分析 docker-compose.yaml

### Redis

项目中用到了`Redis`作为web缓存与`Celery`的消息中间件，此处使用`redis`镜像创建`redis-server`容器。
如果使用外部的`redis-server`可注释掉`Redis`容器配置。

### MySQL

项目的数据库使用`MySQL`，为web服务于`Celery`任务队列提供存储。此处使用`mysql/mysql-server:5.7`镜像创建容器（[查看原因?](/技术/探索docker容器下mysql的数据持久化/)）。
如果使用外部的`mysql-server`可注释掉`MySQL`容器配置。

### Web

web容器负责提供web服务，使用的web框架是`Django`，镜像是基于Ubuntu，安装`Python`，`Mysql-Client`等系统软件与`Django`，`Celery`，`Redis`，`Uwsgi`等`Python`第三方库的自定义镜像。
可以查看我自定义镜像的[Dockerfile](https://github.com/aishenghuomeidaoli/docker-django-mysql-celery/blob/master/web/Dockerfile)，镜像的配置确实完整，但Ubuntu:18.04在配置时区时不能自动通过，与Ubuntu:16.04有区别。
所以不能根据这个Dockerfile打包镜像。我的方法是创建一个Ubuntu:18.04的容器，在容器内安装软件，然后根据容器打包镜像。

指定镜像后，将项目的web目录挂在到容器内，并设置工作目录，在。。。

未完待续

<script>
var _hmt = _hmt || [];
(function() {
  var hm = document.createElement("script");
  hm.src = "https://hm.baidu.com/hm.js?08602829e40aed39037446fe7ea1aa6e";
  var s = document.getElementsByTagName("script")[0];
  s.parentNode.insertBefore(hm, s);
})();
</script>
