---
title: Python--装饰器
date: 2019-05-22 11:04:36
categories:
- 技术
- Python
- 装饰器
tags:
- Python
- 装饰器
- decorator
---

# 背景介绍

我们这里使用现实中饭店中的后厨做类比，一个后厨可能有多个厨师，分别做不同的菜系的菜，比如川菜、鲁菜等。
假设他们在完成一道菜时都有洗菜，切菜，炒菜，摆盘操作，其中不可替代的、最主要的操作是炒菜。

在编程中，有一种情形很常见：在很多函数（不同的厨师）执行（做菜）前/执行后都要执行一些相同/相似的操作（洗菜、切菜、摆盘）。例如：

* 很多web接口视图函数执行前生成用户实例、检验用户登录状态、是否具有某些权限等

* 在执行前，希望将函数的参数通过日志等形式记录下来

* 在执行后，希望处理返回值

<!--more-->

# 代码示例

我们使用`add`, `minus`, `multiply`, `divide`几个函数代表我们在项目中的业务逻辑。

```py
def add(a, b):
    result = a + b
    return result
    
def minus(a, b):
    result = a - b
    return result
    
def multiply(a, b):
    result = a * b
    return result
    
def divide(a, b):
    result = a / b
    return result
```

现在需要在这些函数执行前打印输入参数，在执行后将结果`+1`，并打印。对于一个函数而言，我们很容易想到在函数内增加一些代码，如下面的示例：

```py
def add(a, b):
    print('parameters are %s, %s' % (a, b))
    result = a + b
    result += 1
    print('result is %s' % result)
    return result
```

这种方法达到了目的，但是如果有大量的函数都这样写，无疑耗时又愚蠢。如果后期有变动，比如更改打印的字符串，或者将结果变为`*2`操作，这种方法又不方便统一修改。

类比饭店后厨，可以让厨师只负责炒菜，洗菜、切菜、摆盘的工作可以交给助手（装饰器函数）来处理。
这样的优点是，在厨师炒菜前后（函数执行前后），助手统一控制了厨师（函数）拿到的原材料（输入）和成菜（输出）。
如果老板说菜不切片，要切成丁（控制输入），那只要告诉助手就可以了，不需要告诉每个厨师；对于成菜（输出）的改变也一样适用。

那么回到代码上，关键就在于装饰器函数的编写和使用。

# 装饰器函数

装饰器的原理是重新定义目标函数，重新定义的目标函数具有原来目标函数的逻辑和新增加装饰器内函数的逻辑。这样既增加了额外的逻辑，又不需要改动原来的调用。

由于`Python`内的函数是`一等公民`（函数是实例；函数可以赋值给变量；可以将函数作为参数传递给其他函数；函数可以作为返回值；可以在数据结构中存储函数，如`list`，`dict`、`tuple`等）。

那么就有了下面这种函数：

```py
def my_decorator(func):
    def wrap(*args, **kwargs):
        print(*args)
        result = func(*args, **kwargs)
        result += 1
        return result
    return wrap

def add(a, b):
    result = a + b
    return result

def minus(a, b):
    result = a - b
    return result

add = my_decorator(add)
minus = my_decorator(minus)
```

# 代码分析

下面分析执行过程：

* 重新定义`add`函数。执行`add = my_decorator(add)`时，`my_decorator`返回内部的`wrap`函数，但不执行。

* 调用重新定义的`add`函数。如果执行`add(1, 2)`，相当于执行了`wrap(1, 2)`，`wrap`执行时先执行`print(*args)`，
然后执行`result = func(*args, **kwargs)`，而`func`就是在重新定义是时传入的`add`函数，所以这里会执行原来的`add`函数，并将结果赋值给`result`，之后就进行`+1`操作，然后返回结果。

如果每一个需要装饰的函数都要执行`add = my_decorator(add)`，也一样繁琐，所以`Python`内就有了`@`符号。定义目标函数时，在函数的上一行使用`@` + 装饰器函数的方法可以自动的修改目标函数，和手动重新定义的效果一样。例如，下面这段代码效果和上面的一样：

```py
def my_decorator(func):
    def wrap(*args, **kwargs):
        print(*args)
        result = func(*args, **kwargs)
        result += 1
        return result
    return wrap
    
@my_decorator
def add(a, b):
    result = a + b
    return result

@my_decorator
def minus(a, b):
    result = a - b
    return result
```

# 传递参数

在这个基础上还可以进行扩展，例如，有的函数结果需要`+1`，有的函数结果需要`-2`，那么这样改动

```py
def my_decorator(num):
    def inner_decorator(func):
        def wrap(*args, **kwargs):
            print(*args)
            result = func(*args, **kwargs)
            result += num
            return result
        return wrap
    return inner_decorator
    
@my_decorator(1)
def add(a, b):
    result = a + b
    return result

@my_decorator(-2)
def minus(a, b):
    result = a - b
    return result
```

这和下面的代码是等效的

```
add = my_decorator(1)(add)
minus = my_decorator(-2)(minus)
```

上面的代码可以理解为，`my_decorator`只是为了接收参数，来定义`inner_decorator`。

这样在装饰`add`时，先执行了`my_decorator(1)`，返回了`inner_decorator`，然后由`inner_decorator`装饰`add`函数。

# 保留元信息

参考[《python3-cookbook》9.2 创建装饰器时保留函数元信息](https://python3-cookbook.readthedocs.io/zh_CN/latest/c09/p02_preserve_function_metadata_when_write_decorators.html)

```py
from functools import wraps


def my_decorator_common(func):
    def wrap(*args, **kwargs):
        result = func(*args, **kwargs)
        return result

    return wrap


def my_decorator_wraps(func):
    @wraps(func)
    def wrap(*args, **kwargs):
        result = func(*args, **kwargs)
        return result

    return wrap


@my_decorator_common
def add(a, b):
    """get sum of two parameters

    :param a: 
    :param b: 
    :return: a + b
    """
    result = a + b
    return result


@my_decorator_wraps
def minus(a, b):
    """get difference of two parameters

    :param a: 
    :param b: 
    :return: a - b
    """
    result = a - b
    return result


if __name__ == '__main__':
    print(add.__doc__)
    print(minus.__doc__)
```

上面的示例可以得到如下输出：

```shell
None
get difference of two parameters

    :param a: 
    :param b: 
    :return: a - b
```

可以看到没有使用`wraps`的装饰器会丢失函数的`__doc__`属性。原因就是装饰器的原理是`add = my_decorator(add)`，这个过程没有保留函数的元信息。

如果要保留函数的元信息，比如名字、文档字符串、注解和参数签名等，需要在装饰器内使用`wraps`。这种做法很有必要，比如在编写`Flask`装饰器时。

[Flask Decorator: Login Required Decorator](https://flask.palletsprojects.com/en/1.1.x/patterns/viewdecorators/#login-required-decorator)


# 解除装饰器

对于已经使用了`functools.wraps`装饰器，且只有一个装饰器的函数，可以使用`__wrapped__`方法调用原函数。

详细可参考[《python3-cookbook》9.3 解除一个装饰器](https://python3-cookbook.readthedocs.io/zh_CN/latest/c09/p03_unwrapping_decorator.html)


