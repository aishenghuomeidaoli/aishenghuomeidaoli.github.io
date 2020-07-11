---
title: Python协程与并发——yield
date: 2019-05-14 14:10:07
categories:
- 技术
- Python
- 协程与并发
tags:
- Python
- 协程与并发
- yield
---
首先要说明`yield`的中文释义，“产出；生产”。和`return`不同，`return`是返回，即我们调用一次，就返回一次，而生成器函数的`yield`可以不断的生成值。
举个不恰当的例子，函数返回一个值可以看做是一枚鸡蛋，经过一定的过程，只能孵化出一只小鸡。生成器函数内的`yield`更像一只母鸡（生成器对象），不断地下蛋（产出结果）。

<!--more-->

## 先看`Python2`的用法

```py
In [1]: def get_generator(n):
   ...:     for i in range(n):
   ...:         result = yield (i, i + 1, )
   ...:         print('result is ', result)
   ...:

In [2]: my_generator = get_generator(5)

In [3]: item = my_generator.next()

In [4]: print(item)
(0, 1)

In [5]: item = my_generator.next()
('result is ', None)

In [6]: print(item)
(1, 2)

In [7]: item = my_generator.send('a')
('result is ', 'a')

In [8]: print(item)
(2, 3)

In [9]: item = my_generator.send('b')
('result is ', 'b')

In [10]: item = my_generator.next()
('result is ', None)

In [11]: item = my_generator.next()
('result is ', None)
---------------------------------------------------------------------------
StopIteration                             Traceback (most recent call last)
<ipython-input-11-5d4475abe640> in <module>()
----> 1 item = my_generator.next()

StopIteration:
```

`yield`仅在定义生成器函数时使用，并且需要在生成器函数的函数体内使用。如示例中的`get_generator`函数即为生成器函数。

调用生成器函数时，会返回一个叫做生成器的迭代器。这里需要意识到生成器/迭代器是一个对象。如示例中`my_generator = get_generator(5)`不会执行该函数体内的语句，而是返回了一个生成器对象，赋值给`my_generator`。

执行生成器对象的方法时(`next`, `send`, `throw`, `close`下面会详细说明)，生成器函数体内的代码才会执行。如示例中的第3次输入`In [3]: item = my_generator.next()`，将使`get_generator`函数开始执行。

当执行到`yield`语句时，`yield`语句中的`expression_list`(表达式列表)将作为产出值返回给调用者。如示例中的`(i, i + 1, )`将作为`my_generator.next()`的返回结果，最终赋值给`item`。
同时生成器函数在这里`挂起`/`冻结`（`挂起`是指保留了所有本地状态，包括局部变量，指令指针和内部评估堆栈的绑定关系等，如示例中的局部变量`i`），如示例中的`result = yield (i, i + 1, )`，只会执行`yield (i, i + 1, )`，而不会将`yield`表达式的值赋值给`result`（`yield`表达式的值由下一次调用决定，现在还不存在）。

再次调用生成器对象的方法时，生成器函数会从挂起的地方继续执行，如示例中第5次输入`In [5]: item = my_generator.next()`，`next()`方法会使生成器函数内上一次挂起的`yield`表达式值为`None`，然后执行`result = `的赋值操作。
`yield`语句的执行就像是发起了一次外部调用，会保留函数体内部状态，跳出当前生成器函数，执行外部代码，执行完毕后，继续执行函数体内部的代码。

生成器函数恢复执行时使用的生成器对象方法，决定了`yield`表达式的值，如示例中第5、7次输入`In [5]: item = my_generator.next()`、`In [7]: item = my_generator.send('a')`决定了`yield (i, i + 1, )`表达式的值，分别为`None`和`'a'`（后面详细介绍`next`、`send`方法）。

这些特点使得生成器功能与协程非常相似，可以多次`yield`，有多个入口，执行可被暂停。唯一的区别是生成器函数无法控制`yield`后继续执行的位置，总是生成器的调用者来进行控制。

### 生成器的方法

介绍生成器方法之前，我们先解释一下`当前yield表达式`。生成器函数恢复执行时，从上一次挂起的`yield`表达式的位置执行，这个`yield`表达式叫做`当前yield表达式`，所以在第一次执行生成器函数时，由于不存在上一次挂起，因此不会有`当前yield表达式`的概念，`当前yield表达式`只存在于再次恢复执行中。

生成器的方法可以控制生成器函数的执行，但在生成器对象执行过程中调用这些方法，将会发起`ValueError`异常。

* `generator.next()`

用于启动和恢复执行生成器函数。

使用`next()`方法恢复执行时，会使当前`yield`表达式的值为`None`，并返回下一个`yield`表达式列表的值。
如示例中第5次输入`In [5]: item = my_generator.next()`时，首先从上一次`yield`表达式挂起的位置执行，使`yield (i, i + 1, )`表达式值为`None`，然后赋值给`result`。然后执行`print`，进入下一轮循环，再次`yield`时，表达式列表`(i, i + 1, )`才会返回给当前的`next()`函数。

如果生成器并非以`yield`形式退出，那么调用`next()`方法就会引起`StopIteration`异常，如第11次输入。

* `generator.send(value)`

用于生成器函数的恢复执行，并传递值。

`value`参数会成为当前`yield`表达式的值。如示例中第7次输入`In [7]: item = my_generator.send('a')`，使当前`yield`表达式值为`'a'`。

`send()`方法也会返回下一次`yield`产出的表达式列表，或者在生成器函数没有产出值时，也引起`StopIteration`异常。

如果用`send()`方法启动生成器函数的话，那么`value`参数必须为`None`。因为第一次执行时没有`当前yield表达式`，即没有`yield`表达式可以接收这个传入的值。

```py
In [1]: def echo(value=None):
   ...:     try:
   ...:         while True:
   ...:             yield value
   ...:     except TypeError:
   ...:         yield 'type_error'
   ...:     except GeneratorExit:
   ...:         print('GeneratorExit')
   ...:         if value == 3:
   ...:             yield 3
   ...:         else:
   ...:             raise GeneratorExit
   ...:

In [2]: g1 = echo(1)

In [3]: print(g1.next())
1

In [4]: print(g1.throw(TypeError))
type_error

In [5]: print(g1.next())
---------------------------------------------------------------------------
StopIteration                             Traceback (most recent call last)
<ipython-input-5-ab3315bb99df> in <module>()
----> 1 print(g1.next())

StopIteration:

In [6]: g2 = echo(2)

In [7]: print(g2.next())
2

In [8]: print(g2.close())
GeneratorExit
None

In [9]: print(g2.next())
---------------------------------------------------------------------------
StopIteration                             Traceback (most recent call last)
<ipython-input-9-6591d093f5cb> in <module>()
----> 1 print(g2.next())

StopIteration:

In [10]: print(g2.close())
None

In [11]: g3 = echo(3)

In [12]: print(g3.next())
3

In [13]: print(g3.close())
GeneratorExit
---------------------------------------------------------------------------
RuntimeError                              Traceback (most recent call last)
<ipython-input-12-64ded7703842> in <module>()
----> 1 print(g3.close())

RuntimeError: generator ignored GeneratorExit
```

我们用上面的例子来讲解`throw`和`close`方法。

* `generator.throw(type[, value[, traceback]])`

在生成器挂起的位置发起`type`类型的异常，然后返回下一次生成器产出的值，如果没有产出值则发起`StopIteration`异常。

如示例第2-5次输入中第4次输入`In [4]: print(g1.throw(TypeError))`，会在上一次挂起的位置即`yield value`处发起`TypeError`异常，然后被`except TypeError:`捕获，然后执行`yield 'type_error'`，产出值返回给调用者，所以第4次输入会打印出`type_error`。

如果生成器函数没有捕捉到传入的异常，或者发起了其他异常，那么该异常会向上抛出至调用者。

* `generator.close()`

在生成器挂起的位置发起`GeneratorExit`异常。

如果生成器函数发起`StopIteration`(生成器函数执行完成或执行`close()`方法都会使生成器结束，再次获取产出值时会发起`StopIteration`异常)或`GeneratorExit`(当该异常未被捕获时)异常，那么生成器就会关闭，然后返回至调用者，并且代码不会发起异常。

如上面示例的第6-10次输入。`In [6]: g2 = echo(2)`创建了一个生成器。`In [7]: print(g2.next())`执行生成器，并在`yield value`处挂起。
`In [8]: print(g2.close())`调用`close()`方法，这时会在上次挂起，即`yield value`处发起`GeneratorExit`异常，所以被`except GeneratorExit:`捕捉到，然后生成器函数执行`print('GeneratorExit')`，我们可以看到输出。
生成器传入的参数为2，所以接下来会执行`raise GeneratorExit`，从执行结果来看，`close()`方法触发生成器函数发起的`GeneratorExit`，不会传递至调用者，也没有返回值。
继续获取生成器产出值`In [9]: print(g2.next())`，会发起`StopIteration`，证明生成器已经被停止了。
`In [10]: print(g2.close())`继续执行`close()`方法，会在生成器挂起的位置继续执行，但此时生成器已经停止了，会发起`StopIteration`异常，示例中输出`None`，表明`close()`方法触发生成器函数发起的`StopIteration`异常，不会传递至调用者，也没有返回值。

已关闭的生成器继续返回产出值会发起`RuntimeError`异常。如示例中第10-13次输入。`close()`方法会被`except GeneratorExit:`捕获，根据逻辑会执行`yield 3`语句，此时报错。

生成器内发起的其他异常会传递至调用者。如果生成器由于异常结束，或执行完毕结束时，调用`close()`不会有任何影响。

## 先看`Python3`的用法

这里只说相比`Python2`新增的内容。

`python3`中可以使用`yield from expression`表达式，并且除了可以在定义生成器函数时使用`yield`表达式，也可以在定义异步生成器函数（`async def`定义的函数体内使用`yield`）时使用。

### 生成器函数

`try`结构中可以使用`yield`表达式。如果生成器在结束（引用计数为0或被垃圾回收）之前没有恢复执行，那么生成器的`close()`方法将被调用，并执行`finally`语句。

`yield from <expr>`语句将表达式当做子迭代器使用。所有子迭代器产出的值，都会传递给当前生成器方法的调用者。
`send()`方法传入的值、`throw()`传入的异常都会传递至底层的迭代器（子迭代器），前提是底层的迭代器有对应的合适的方法，否则，`send()`方法会引起`AttributeError`或`TypeError`异常，`throw()`方法会立即抛出传入的异常。

底层迭代器完成之后，发起的`StopIteration`对象的`value`属性就会成为当前`yield`表达式的值。`yield`表达式的值可以由`StopIteration`异常精确设置，也可以在子迭代器是生成器时（子生成器会产出值）自动设置。



### 异步生成器函数


