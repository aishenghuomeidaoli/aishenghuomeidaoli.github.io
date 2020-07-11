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
所以不能根据这个Dockerfile打包镜像。我是创建Ubuntu:18.04的容器，安装软件后，根据容器打包镜像。

指定镜像后，使用`volumes`参数将项目的web目录挂在到容器内，并使用`working_dir`参数设置工作目录。
由于项目中HaProxy进行了负载均衡代理，所以并不需要暴露端口，如需映射端口到宿主机可以将`ports`参数设置为8000:8000。

在web容器中需要链接`mysql`容器与`redis`容器中的服务，所以用`depends_on`制定依赖的容器，这样会先启动`mysql`容器与`redis`容器，再启动`web`容器，并且在`web`中使用'mysql'、'redis'即可连接到对应的服务。

在web项目中有一些关键的配置参数，我们可以通过环境变量传入，这样通过docker-compose.yml文件即可管理项目配置。此处我们传入了，Django项目秘钥，MySQL的服务主机地址即'mysql'字符串，与mysql容器一致的root用户密码，创建好的数据库名称，Redis服务主机地址即'redis'字符串。

最后是容器启动后执行的命令：

```
sh ./run.sh
```

因为将`web`目录挂在到了容器内，并且工作目录与项目目录相同，所以`run.sh`脚本在容器的当前目录下。我们来看下次脚本：

```
python3 manage.py makemigrations
python3 manage.py migrate
uwsgi --ini /usr/src/app/web/uwsgi.ini
tail -f /usr/src/app/logs/uwsgi.log
```

因为不知道是否是第一次运行容器，所以加了迁移命令，第一次运行时会在数据库中创建所需的表。之后运行也不会引起错误。

然后使用uwsgi启动项目。最后监听uwsgi日志文件，并输出到终端，这样才不会使容器退出。

### Celery

`celery`任务队列的代码是写在Django项目中的，使用了web容器相同的环境与配置。唯一的区别是容器启动后的运行命令。

### HaProxy

`haproxy`是一个性能极高的代理软件，我们在这里用作负载均衡的转发工作。这里使用了[dockercloud/haproxy](https://hub.docker.com/r/dockercloud/haproxy/)镜像，将容器80端口映射至宿主机的80端口，并指定连接的容器名`web`，更多用法可以查阅dockercloud/haproxy镜像页面描述。

## 启动项目

### 初始化MySQL

这里会创建一个`MySQL`数据库，方便`Django`服务端连接。

```
docker-compose -f docker-compose-mysql.yaml up -d

mysql -u root -h 0.0.0.0 -p
```

执行SQL，创建数据库，修改时区。

```
CREATE DATABASE mydb CHARACTER SET utf8 COLLATE utf8_general_ci;
SET time_zone = 'Asia/Shanghai';
```

如果你有一个远程的`MySQL`服务器，可将docker-compose.yaml中mysql容器及对mysql容器的依赖注释掉，并在docker-compose.yaml中将web容器与celery容器环境变量参数`environment`中`MYSQL_HOST`的数值修改为远程MySQL的IP。

### 启动项目

这里会为 mysql、web、redis、haproxy创建容器。

```
docker-compose up -d

```

因为`Haproxy`的存在，可以开启多个web容器，支持负载均衡。

```
docker-compose up -d --scale web=2
```

### 访问页面

打开浏览器，访问 http://0.0.0.0/ ，创建一个运算任务、任务执行结束后，页面会刷新任务列表。


### 删除所有容器

```
docker-compose down
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
