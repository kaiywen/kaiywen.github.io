---
layout:     post
title:      "Python 定义迭代器"
subtitle:   ""
date:       2016-4-3 10:02:00
author:     "YuanBao"
header-img: "img/post-python.jpg"
header-mask: 0.25
catalog: true
tags:
 - python
 - programming
---

在大部分的编程语言中，我们都要接触迭代器的概念。迭代器（iterator）有时又称游标（cursor），它提供了一种遍历容器的接口，使得使用者无需关心容器的内部实现。在 Python 中，凡是可以使用 `for` 遍历的结构（例如 `list`, `dict`, `str`, `tuple`），全部都提供了迭代器接口。然而，我们经常需要创建一些新的类型或者结构，并让它支持 `for` 语句的遍历。这时候我们需要为新的类型实现一个自定义的迭代器接口。

## 定义迭代器
在 Python 中为一个类添加迭代器共需三个步骤：

1. 为新类添加一个`__iter__()` 方法，该方法返回迭代器本身。
2. 为新类添加一个 `next()`（在 python3 中是`__next__()`）方法，该方法返回容器中的下一个元素，并且在超出下标范围时抛出 `StopIteration` 异常。

<!--more-->

通过下面一个简单的例子展示一下具体的实现:

```python
class Container(object):
    def __init__(self, begin, end):
        self.begin = begin
        self.end = end
	
    def __iter__(self):
        self.begin = 0
        return self
	
    def next(self):
        if self.begin <= self.end:
            index = self.begin
            self.begin += 1
            return index
        else:
            raise StopIteration()


container = Container(0, 3)
for num in container:
	print num
```
如果我们能够了解 `for` 语句在遍历过程中的作用，就能更加清楚地明白迭代器的工作原理。事实上上述代码中的 `for` 语句等同于下面的几行代码:

```python
itr = container.__iter__()
while True:
    try:
        x = itr.next()
    except StopIteration:
        break
    """
    Do someting related to x
    """
```

## 生成器

我们也可以通过定义生成器的方式来实现迭代器的功能（有关生成器的细节后续再说），生成器允许我们轻松地实现一个在获取下一个值之前不占用空间的结构。这里我们可以使用 `yield` 关键字来定义一个生成器，例如:

```python
def generator(index, end):
    while index < end:
        yield index
        index += 1
g = generator(0, 5)
print next(g)
print next(g)
```

这时候 g 作为一个生成器也具有了可迭代的接口，我们同样可以使用 `next` 函数或者 `for` 循环来对 g 进行遍历。

