---
title: Mixin
date: 2019-04-16 09:10:13
categories:
- 技术
- Python
- Python高级编程
tags:
- Python
---

参考资料：

[Mixins for Fun and Profit](https://easyaspython.com/mixins-for-fun-and-profit-cb9962760556)

## 介绍

写代码最好能够复用代码，避免重复造轮子。在面向对象的语言中，有一个概念叫做`Mixin`，在Python中通过多重继承来实现。

不同的语言中，`Mixin`形式会有区别，但是相同的是，他们都封装了可重用的代码，以供其他类使用。

真正的继承和`Mixin`的区别很微妙，`Mixin`不会单独使用，并且也不作为抽象类。

<!--more-->

## 示例

对已有代码添加日志功能，可以用下面的方法实现：

```python
import logging

class EssentialFunctioner(object):
    def __init__(self):
        ...
        
        self.logger = logging.getLogger(
            '.'.join([
                self.__module__,
                self.__class__.__name__
            ])
        )
    def do_the_thing(self):
        try:
            ...
        except BadThing:
            self.logger.error('OH NOES')
```

上面的方式不算差，但是如果有几十个类，就会变得复杂。可以创建`LoggerMixin`类，达到相同的效果：

```python
import logging

class LoggerMixin(object):
    @property
    def logger(self):
        name = '.'.join([
            self.__module__,
            self.__class__.__name__
        ])
        return logging.getLogger(name)
```

封装好日志模块后，可以添加至已有的代码中：

```python
class EssentialFunctioner(LoggerMixin, object):
    def do_the_thing(self):
        try:
            ...
        except BadThing:
            self.logger.error('OH NOES')

class BusinessLogicer(LoggerMixin, object):
    def __init__(self):
        super().__init__()
        self.logger.debug('Giving the logic the business...')
```

这样只需要编写一次代码，在使用时继承`LoggerMixin`，调用`self.logger`即可。

`Mixin`的重度使用可以节约很多时间，并且提升代码的可阅读性。

该作者的一个使用案例是，在一个`Django`项目中，有几个基于类的视图函数需要控制职能由内网发起访问。有一个中间件可以进行检查，如果通过外网访问直接响应`HTTP 404`，那么就需要在视图函数的`dispatch`方法上使用`decorator_from_middleware`和`method_decorator`。

```python
from django.utils.decorators import decorator_from_middleware, method_decorator
from django.views import generic
from app import middleware

network_protected = decorator_from_middleware(
    middleware.NetworkProtectionMiddleware
)

@method_decorator(network_protected, name='dispatch')
class SecretView(generic.View):
    ...
```

如果有其他的视图函数需要进行控制，那么也需要为他们添加装饰器。作者的做法是，将网络保护的代码集成到`Mixin`，允许基于类的视图函数继承：

```python
from django.utils.decorators import decorator_from_middleware, method_decorator
from django.views import generic

network_protected = decorator_from_middleware(
    middleware.NetworkProtectionMiddleware
)

class NetworkProtectionMixin(object):
    @method_decorator(network_protected)
    def dispatch(self, *args, **kwargs):
        return super().dispatch(*args, **kwargs)

class SecretView(NetworkProtectionMixin, generic.View):
    ...
```

同理，可以将`Mixin`用于权限、认证等方面。`Mixin`使代码更具表现力，降低了复杂性，方便了理解和单元测试。


