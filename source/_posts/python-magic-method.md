---
title: PYTHON类中的“魔法”方法
description: 一些常用的magic method的用法及区别
date: 2018-09-10
toc: true
categories: python
tags:
  - magic method
  - 类方法
---

#### 一、常见魔法方法


python类中有一类特殊内建方法，可以通过在类中定义这些方法，实现类似于装饰器一样的行为，改变类本身的默认行为，python称这些行为“magic method”

常见的“magic method”有如下几种：
- `__str__` 与 `__repr__`
- `__getattr__` 与 `__getattribute__`
- `__setattr__` 与 `__delattr__`
- `__iter__` 与 `__getitem__`
- `__init__` 与 `__new__`

#### 二、`__str__` 与 `__repr__`

`__str__`相当于str()方法的调用，`__repr__`相当于repr()方法调用。str是针对于让人更好的理解的字符串格式化，主要适用于定义对类实例直接进行print()方法时输出；而repr是让机器更好理解的字符串格式化，定义在IDE中直接调用类是输出的字符串

```python
>>> class A():
...     def __str__(self):
...         return 'instance of class A'
...     def __repr__(self):
...         return 'instance of class A in memory'
...
>>> a = A()
>>> a
instance of class A in memory
>>> print(a)
instance of class A
```

#### 三、`__getattr__` 与 `__getattribute__`

python手册中对这两种方法的描述如下：
> `object.__getattr__(self, name)`
> Called when the default attribute access fails with an AttributeError (either `__getattribute__()` raises an AttributeError because name is not an instance attribute or an attribute in the class tree for self; or `__get__()` of a name property raises AttributeError). This method should either return the (computed) attribute value or raise an AttributeError exception.Note that if the attribute is found through the normal mechanism, `__getattr__()` is not called. (This is an intentional asymmetry between `__getattr__()` and` __setattr__()`.) This is done both for efficiency reasons and because otherwise` __getattr__()` would have no way to access other attributes of the instance. Note that at least for instance variables, you can fake total control by not inserting any values in the instance attribute dictionary (but instead inserting them in another object). See the `__getattribute__()` method below for a way to actually get total control over attribute access.

> `object.__getattribute__(self, name)`  
> Called unconditionally to implement attribute accesses for instances of the class. If the class also defines `__getattr__()`, the latter will not be called unless` __getattribute__()` either calls it explicitly or raises an AttributeError. This method should return the (computed) attribute value or raise an AttributeError exception. In order to avoid infinite recursion in this method, its implementation should always call the base class method with the same name to access any attributes it needs, for example, `object.__getattribute__(self, name)`.

上述描述表示，当用户在调用实例属性时，会首先无条件调用getattribute方法查找其属性并返回对应的value。  

而getattr方法只有在访问实例属性失败时，即getattribute方法查找失败时才会调用，一旦实例属性被定义，再次查找属性会由getattribue提供返回值，不再调用getattr方法
```python
class Person:
    def __getattr__(self, attr):
        print('call __getattr__')
        if attr == 'name':
            return 'lol no name'
        elif attr == 'age':
            return 'don\'t tell U'

    def __getattribute__(self, attr):
        print('call __getattribute__')
        # return self.name  # self.name会调用__getattribute__方法，这个返回会导致无限嵌套调用。
        return super(Person, self).__getattribute__(attr)

s1 = Person()
print(s1.name)
print('---')
s1.name = 'Maria'
print(s1.name)

运行结果：
call __getattribute__    # 查找实例属性时，先调用__getattribute__查找
call __getattr__         # __getattribute__查找失败后调用__getattr__
lol no name
---
call __getattribute__     
Maria
```
#### 四、`__setattr__` 与 `__delattr__`

`__setattr__`与`__delattr__`分别在对实例属性进行设置和删除的时候调用，在重写这类方法时需要注意上述例子中无限嵌套循环调用。

```python
class Person()：
    def __setattr__(self, name, value):
        print('call __setattr__')
        # self.name = value  # 这里对实例属性name设置会调用__setattr__方法，导致无限循环嵌套调用，因此不能使用这种方式。
        super(Person, self).__setattr__(name, value)
    def __delattr__(self,name):
        print('call __delattr__')
        super(Person, self).__delattr__(name)
    def __init__(self, name):
        self.name = name

a = Person('Maria')
print('---')
a.age = 28
print('---')
del a.age

运行结果：
call __setattr__      # 实例初始化时，__init__函数中的初始化属性也调用__setattr__
---
call __setattr__
---
call __delattr__
```

#### 五、`__iter__` 与 `__getitem__`

`__iter__`返回一个迭代器，可用于对类实例进行迭代，可通过for循环遍历；`__getitem__`用于实现对实例以*dict[item]*或者*list[index]*的方式获取类属性，即*`x.__getitem__(y) <=> x[y]`*。

当类仅实现了`__getitem__`方法而没有实现`__iter__`时，for循环会默认初始化一个int类型的index，默认由0开始，并作为`__getitem__`的参数，若`__getitem__`重写为可以通过这个index找到相应的属性值，则此时没有`__iter__`方法的实例也可用于for循环，但导入collections模块中的Iterable和Iterator之后使用isinstance判断发现，此时实例即不是Iterable也不是Iterator

```python
from collections import Iterable, Iterator
class Person():
    def __init__(self):
        self.names = ['Maria', 'Miya', 'Crystal']
        self.data = {'Maria': 28, 'Miya': 27, 'Crystal': 18}
    def __iter__(self):
        print('__iter__ called')
        return iter(self.data.values())
    def __getitem__(self, index):
        print('__getitem__ called')
        if isinstance(index, int):
	        return self.names[index]
        elif isinstance(index, str):
            return self.data[index]
    
class People():
    def __init__(self):
        self.name = ['Maria', 'Miya', 'Crystal']
        self.data = {'Maria': 28, 'Miya': 27, 'Crystal': 18}
    def __getitem__(self, index):
        print('__getitem__ called')
        if isinstance(index, int):
	        return self.names[index]
        elif isinstance(index, str):
            return self.data[index]

s1 = Person()
print('----1----')
print(s1[1], s1['Miya'])
print('----2----')
s2 = People()
print('----3----')
print(s2[2], s2['Crystal'])
print('----4----')
print(isinstance(s1, Iterable))
print(isinstance(s1, Iterator))
print('----5----')
print(isinstance(s2, Iterable))
print(isinstance(s2, Iterator))
print('----6----')
for i in s1:
    print(i)
print('----7----')
for i in s2:
    print(i)

    
执行结果：
----1----
__getitem__ called     # s1[1]调用一次 __getitem__
__getitem__ called     # s1['Miya']调用一次 __getitem__
Miya 27
----2----
----3----
__getitem__ called
__getitem__ called
Crystal 18
----4----
True					# 实例s1实现了__iter__方法，所以实例s1是可迭代的对象
False					# 实例s1本身并不是一个迭代器，__iter__方法应该返回为一个迭代器
----5----
False					# 实例s2没有实现__iter__方法，实例s2不是一个可迭代的对象
False					# 实例s2也不是一个迭代器
----6----
__iter__ called			# 实例s1实现了__iter__方法，对实例使用 for ... in 循环
28						# 会自动调用__iter__对实例对象迭代
27
18
----7----
__getitem__ called		# 实例s2没有实现__iter__方法，对实例调用 for ... in 循环
Maria					# 每次循环传入一个index，调用__getitem__,默认从0开始,并
__getitem__ called		# 递增1。由于index为int类型，根据代码每次返回一个name，第四
Miya					# 次循环时，传入index为3，已超出index范围，调用__getitem__
__getitem__ called		# 会返回IndexError,导致循环退出。因此结果有3个返回值，4次调用
Crystal					# __getitem__的记录。
__getitem__ called
```



#### 六、``__init__`` 与 `__new__`

这两个方法在实例初始化时按顺序调用，当初始化一个实例时，解释器先调用`__new__()`生成一个实例对象，然后根据`__init__()`来初始化实例。通过如下例子观察两个函数的执行顺序：

```python
class Person():
    def __init__(self, name, age):
        print('__init__ called')
        self.name = name 
        self.age = age
    def __new__(cls,*args,**kwargs):
        print('__new__ called')
        return super(Person,cls).__new__(cls)

s1 = Person()

运行结果：
__new__ called
__init__called
```

