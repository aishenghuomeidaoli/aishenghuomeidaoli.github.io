---
title: Python3 队列
date: 2019-04-03 18:19:27
tags:
---

## `collections.deque`

`deque`提供了`popleft`、`appendleft`等函数

<!--more-->

```python
from collections import deque

q = deque([1, 2, 3])

len(q)  # 3

q.append(1)  # deque([1, 2, 3, 1])

q.appendleft(4)  # deque([4, 1, 2, 3, 1])

right = q.pop()  # right == 1, q == deque([4, 1, 2, 3])

left = q.popleft()  # left == 4, q == deque([1, 2, 3])

q.extend([9, 8])  # q == deque([1, 2, 3, 9, 8])

q.extendleft([1, 2])  # q == deque([2, 1, 1, 2, 3, 9, 8]) 这里注意被extend的list是倒序插入的

q.count(1)  # 2

q.index(3)  # 4

q.insert(1, 5)  # q == deque([2, 5, 1, 1, 2, 3, 9, 8]) 指定索引插入元素

q.remove(1)  # q == deque([2, 5, 1, 2, 3, 9, 8]) 移除第一个匹配的元素

q.reverse()  # q == deque([8, 9, 3, 2, 1, 5, 2])

q.rotate()  # q == deque([2, 8, 9, 3, 2, 1, 5])

q.rotate(3)  # q == deque([2, 1, 5, 2, 8, 9, 3])  将结尾三个元素旋转至开始
```