---
title: python的赋值和参数传递
date: 2018-12-24 20:26:13
tags:
  - python
  - 面试
---

最近在看python的面试题的时候，遇到这个问题“python是值传递还是引用传递”，因此记录下我对这个问题的学习和思考。

<!-- more -->

## 一切皆对象

在python里，一切皆对象（Everything in Python is an object）。

数字、字符串、元组、列表、字典、函数、方法、类、模块等等都是对象。

```python
>>> type(3)
<class 'int'>
>>> type(3.2)
<class 'float'>
>>> type('hello')
<class 'str'>
>>> type(())
<class 'tuple'>
>>> type([])
<class 'list'>
>>> type({})
<class 'dict'>
>>> def func():
...     pass
...
>>> type(func)
<class 'function'>
>>> class A:
...     def method(self):
...         pass
...
>>> type(A)
<class 'type'>
>>> type(A.method)
<class 'function'>
>>> import sys
>>> type(sys)
<class 'module'>
>>> type(type)
<class 'type'>
```

## 对象的特性

python的所有对象都有3个特性：

-   身份标识
-   类型
-   值

### 身份标识

python有一个内置函数[id](https://docs.python.org/3/library/functions.html?highlight=id#id)

>`id`(*object*)
>
>Return the “identity” of an object. This is an integer which is guaranteed to be unique and constant for this object during its lifetime. Two objects with non-overlapping lifetimes may have the same [`id()`](https://docs.python.org/3/library/functions.html?highlight=id#id) value.
>
>**CPython implementation detail:** This is the address of the object in memory.

在一个对象的生命周期内，对它调用id函数，会得到一个唯一的值。如果两个对象的生命周期不重叠的话，那么它们的id值可能会相同。在CPython的实现细节中，id函数得到的就是这个对象在内存中的地址。

### 类型

调用type()可以得到对象的类型，当然type也是一个特殊的对象。

### 值

可以理解为如何根据类型来解释这块内存区域，从而得到数据值

## 赋值的实质

在我看来，python的赋值就是名称的绑定，把一个变量名绑定到某个对象上。

对于不可变类型，在内存中只有1个备份。比如下面的例子：

```python
a = 3
b = a
c = int(3)


def func(d):
    print(id(d))


print(id(a), id(b), id(c))
func(a)
```

```
1440339792 1440339792 1440339792
1440339792
```

这样在赋值的时候就可以不用创建新的对象，直接把新的变量名绑定到原来的对象上就行了。每当有一个新的变量指向一个对象时，它的引用计数就会加1。当引用计数变为0后，python就会回收它占用的内存。

## 回归到问题本身

有的人认为，python在传递不可变类型的时候是值传递，而在传递可变类型的时候是传递引用。他们的依据是：传递不可变类型时，函数内部改变这个变量的值，不能影响函数外的值。如下面的例子：

```python
def change(b):
    b = 2


a = 1
print('before:', a)
change(a)
print('after:', a)
```

```
before: 1
after: 1
```

而在我看来，这只是从表象上来看的观点，而实质是，在调用函数`change`的时候，执行了`b=1`，这个时候变量`b`绑定到了`int`型、值为`1`的对象上，然后在函数内部的`b=2`根本没有试图去改变外面的变量`a`，它只是把变量`b`绑定到了`2`这个`int`型变量上了。

-   参数传递之前，`int`型、值为`1`的对象的引用计数为1
-   参数传递之后，`int`型、值为`1`的对象的引用计数为2

那为什么在函数内部可以改变可变类型呢，其实这就很好理解了，比如下面的代码：

```python
def change(b):
    b.append(2)


a = [1]
print('before:', a)
change(a)
print('after:', a)
```

```
before: [1]
after: [1, 2]
```

在参数传递之后，`b`和`a`指向同一个`list`对象，那么`b.append`其实是等同于`a.append`的，都是对内存中同一个对象的`append`方法的调用而已。如果在函数中执行`b=3`，也丝毫无法影响外面的变量`a`。

## 总结

所以，在我看来，python的参数传递是引用传递。参数传递前后，某个对象的引用计数增加了1。