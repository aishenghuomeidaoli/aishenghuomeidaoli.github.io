---
title: 探索Docker容器下MySQL的数据持久化
date: 2018-07-31 16:37:25
categories:
- 技术
tags:
- Docker
- MySQL
description: 探索Docker容器下MySQL的数据持久化
keywords: 
- Docker
- Docker Compose
- 容器
- MySQL
abstract: true
excerpt: true
---

* [背景介绍](/技术/探索docker容器下mysql的数据持久化/#背景介绍)

* [Docker容器下MySQL数据持久化](/技术/探索docker容器下mysql的数据持久化/#Docker容器下MySQL数据持久化)

## 背景介绍

容器化技术为开发带来了巨大的便捷，`Docker`作为容器化技术的典型代表，广泛的应用在工程生产中。
同时，以`Docker`为基础的其他技术，也极大地丰富了`Docker`的功能，如`Python`编写的容器编排技术`Docker Compose`，谷歌开源的容器管理技术`Kubernetes`，都为传统的开发、部署、维护的工作打开了新世界的大门。

最近打算用`docker-compose`创建一个 网页客户端（web）+ 服务端接口（api）+ 任务队列（celery）的项目。
按照传统的部署方式，需要在宿主机上启动的进程有：数据库（MySQL/MongoDB）、缓存（Redis/RabbitMQ/..）、任务队列（celery）、后端API（Python/..）、HTTP服务器（Nginx/Apache）。缺点有一些：初次部署环境配置较多、多个进程维护成本高。

如果采用容器的方式部署，首先解决了本地开发与生产环境差异产生的影响；其次，采用`Docker Compose`或`Kubernetes`更加符合模块化开发的风格，降低了耦合性，提高了可维护性。

最近在看`Docker Compose`与`Kubernetes`，其中容器相互引用利用的是容器名，在创建容器时将指定的容器名写入`hosts`，这样就可以起到和**localhost**一样的效果。
但如果在宿主机上运行的进程，比如`mysql-server`或内部其他接口，是不能从容器内进行调用的，容器的端口映射只能将端口进行**暴露**，只能在宿主机内单向连通。

对此，我想到的解决有两个：一个是通过公网IP进行连接；另一个就是将这些服务进程运行在容器内。

第一个办法这无疑降低了性能，提高了网络延迟，不可取。第二个办法对于一般的服务可行，但是对于数据库需要考虑数据持久化，不能因为容器停止、重启等造成数据的丢失。

<!--more-->

## Docker容器下MySQL数据持久化

### 前提

`mysql-server`镜像要比`mysql`镜像小几百兆，更为精简；MySQL 8.0版本在用户认证机制上做了变动，在探索过程中没有找到较好的解决办法，只好选择 MySQL 5.7版本。

### 卷挂载

根据博客[Docker MySQL Persistence](http://blog.arungupta.me/docker-mysql-persistence/)，与[mysql/mysql-server镜像文档](https://hub.docker.com/r/mysql/mysql-server/)的内容。

`MySQL`支持数据的持久化，只要将指定的宿主机目录挂在到`MySQL`默认数据存储路径 */var/lib/mysql*中即可。

```$xslt
docker run -it --name mysqldb \
-v /path/to/local-mysql:/var/lib/mysql \
-e MYSQL_ROOT_PASSWORD=password \
-d mysql/mysql-server:5.7
```

将此目录以相同方式挂载到其他容器中，即可以实现`Docker`容器下`MySQL`数据的持久化，但被挂载的目录只能被一个`MySQL`容器使用，不能多个`MySQL`容器同时挂载。

```$xslt
docker run -it --name mysqldb \
-v /path/to/local-mysql:/var/lib/mysql \
-e MYSQL_ROOT_PASSWORD=password \
-d mysql/mysql-server:5.7
```

### 开启远程访问

在解决数据持久化之后，需要解决连接问题。

* 首先是暴露端口，增加`-p`参数：

```$xslt
docker run -it --name mysqldb \
-v /path/to/local-mysql:/var/lib/mysql \
-e MYSQL_ROOT_PASSWORD=password \
-p 3306:3306 \
-d mysql/mysql-server:5.7
```

* 其次是远程访问，需要将指定用户的访问地址`host`设为访问IP或`%`，这样可以从宿主机，或其他容器连接。

如果使用的是`root`用户，则需设置`MYSQL_ROOT_HOST`环境变量：

```$xslt
docker run -it --name mysqldb \
-v /path/to/local-mysql:/var/lib/mysql \
-e MYSQL_ROOT_PASSWORD=password \
-e MYSQL_ROOT_HOST=% \
-p 3306:3306 \
-d mysql/mysql-server:5.7
```

也可以创建新用户，此时需要额外的三个环境变量：用户名`MYSQL_USER`、用户密码`MYSQL_PASSWORD`、授权该用户的数据库名`MYSQL_DATABASE`，此时创建的用户的访问`host`为 *%*，即任意IP：

```$xslt
docker run -it --name mysqldb \
-v /path/to/local-mysql:/var/lib/mysql \
-e MYSQL_USER=new_username \
-e MYSQL_PASSWORD=new_user_password \
-e MYSQL_DATABASE=user_database \
-e MYSQL_ROOT_PASSWORD=password \
-p 3306:3306 \
-d mysql/mysql-server:5.7
```

<script>
var _hmt = _hmt || [];
(function() {
  var hm = document.createElement("script");
  hm.src = "https://hm.baidu.com/hm.js?08602829e40aed39037446fe7ea1aa6e";
  var s = document.getElementsByTagName("script")[0];
  s.parentNode.insertBefore(hm, s);
})();
</script>
