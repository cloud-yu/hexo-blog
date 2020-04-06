---
title: PYTHON闭包中的延迟绑定
description: 记录PYTHON中闭包延迟绑定的效果，以及如何避免
date: 2018-08-02
toc: true
tags:
  - 闭包
  - 延迟绑定
  - lazy binding
categories: python
---

#### 一、什么是闭包

先给出闭包的字面定义：**闭包是由函数及其相关的引用环境组合而成的实体本质上是一个能读取其他函数内部变量的函数**。

如果在一个函数的内部定义了另一个函数，外部的我们叫他外函数，内部的我们叫他内函数。闭包：在一个外函数中定义了一个内函数，内函数里运用了外函数的临时变量，并且外函数的返回值是内函数的引用。这样就构成了一个闭包。

一般情况下，如果一个函数结束，函数内部所有东西都会释放，局部变量会消失。但是闭包不同，如果外函数在结束时发现自己的临时变量会在内部函数中用到，就把这个**临时变量绑定给了内函数**，然后再自己结束。

#### 二、python闭包延迟绑定（late binding）示例

先来看一段代码：

```python
def testfunc():
    temp = [lambda x: i*x for i in range(4)]
    return temp

# expected result： 0 2 4 6
for everyfunc in testfunc():
    print(everyfunc(2))  

>>>
6
6
6
6
>>>
```
如上一段代码，原本预期得到结果为{0 2 4 6}，实际得到结果却是{6 6 6 6}，原因是闭包函数内部代码只有在函数调用时才会对自由变量进行绑定查找。

为了方便解释，我们将原有代码的lambda函数改写一下：

```python
def testfunc():
    temp = []
    for i in range(4):
        def f(x):
            return i * x
        temp.append(f)
    return temp

for everyfunc in testfunc():
    print(everyfunc(2))
```
如上，闭包返回内部函数 f 的一个序列，每一个内部函数 f 都包含自由变量 i，和一个形参 x，即 testfunc 返回的序列为 

```python
return [ lambda x: x*i, lambda x: x*i, lambda x: x*i, lambda x: x*i]
```

这个**自由变量 i 在函数 everyfunc 被调用时才会进行查找绑定**，此时外部函数 testfunc中的 for 循环已经执行完毕，当前 $i = 3$  ,于是每一个 everyfunc(2) 被调用时绑定的 $i = 3$，函数执行结果为 $2 \times 3 = 6$ 。  

#### 三、如何解决延迟绑定导致代码结果与预期不符的问题

##### 3.1 将外部函数循环中的 i 的值，以形参初始化值的方式传给内部函数。

在每次循环时，内部函数中的一个形参初始化值为当前循环下 i 的值，相当于给每个函数一个默认参数。

```python
def testfunc():
    temp = [lambda x, i=i: i*x for i in range(4)]
    return temp
# 上下两种写法效果一致
def testfunc():
    temp = []
    for i in range(4):
        def f(x, i=i):
            return x * i
        temp.append(f)
    return temp
```

##### 3.2 使用python自带 functools.partial 模块，锁定 i 值

functools.partial 这个函数可接受多个参数，按照顺序，第一个参数为一个函数`func`，后续参数可以是`args`或者`kwargs`，其中`args`表示将`func`中的`args`参数按从左往右的顺序固定。

```python
from functools import partial
def testfunc():
    temp = [partial((lambda x,i:x*i),i) for i in range(4)]
    return temp
# 或者
def testfunc():
    temp = []
    for i in range(4):
        def funct1(i,x):
            return x * i
        f = partial(funct1, i)
        temp.append(f)
    return temp
```

