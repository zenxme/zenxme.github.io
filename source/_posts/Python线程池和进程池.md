---
title: Python线程池和进程池
date: 2019-04-11 16:15:36
tags:
  - python
  - 线程
  - 进程
---

Python3.2之后，标准库里引入了`concurrent.futures`模块，为异步调用提供了高级的接口。在此记录下我对其中的`ThreadPoolExecutor`和`ProcessPoolExecutor`类的学习和理解。

<!-- more -->

## ThreadPoolExecutor

`ThreadPoolExecutor`是`Executor`类的子类。它有一个参数是`max_workers`，指定了线程池中最多同时执行的线程数量。这个是最常用的参数，3.5版本之后又增加了几个参数，但是不常用，可以去文档里查看。

### 1个线程

开启1个线程访问`https://www.baidu.com/`

```python
from concurrent.futures import ThreadPoolExecutor

import requests


def get_headers(url):
    return (url, requests.get(url).headers)


with ThreadPoolExecutor(1) as executor:
    f = executor.submit(get_headers, 'https://www.baidu.com/')
    print(f.result())
```

### 3个线程

开启3个线程分别访问`https://www.baidu.com/`、`https://www.163.com/`、`https://www.qq.com/`

```python
from concurrent.futures import ThreadPoolExecutor, as_completed

import requests


def get_headers(url):
    return (url, requests.get(url).headers)


with ThreadPoolExecutor(3) as executor:
    fs = [
        executor.submit(get_headers, 'https://www.baidu.com/'),
        executor.submit(get_headers, 'https://www.163.com/'),
        executor.submit(get_headers, 'https://www.qq.com/'),
    ]
    for f in as_completed(fs):
        print(f.result())
```

也可以使用`map`函数

```python
from concurrent.futures import ThreadPoolExecutor

import requests


def get_headers(url):
    return (url, requests.get(url).headers)


with ThreadPoolExecutor(3) as executor:
    urls = [
        'https://www.baidu.com/',
        'https://www.163.com/',
        'https://www.qq.com/',
    ]
    for result in executor.map(get_headers, urls):
        print(result)
```

需要注意的是，使用`map`函数时，返回的结果的顺序和Python内置的map函数一样，是按照传递进去的顺序来的，并不是真正的它们完成的顺序。比如在上面的代码里，第一个返回的永远是baidu的请求。通过查看源码也印证了这一问题。

`Executor.map`的源码

```python
    def map(self, fn, *iterables, timeout=None, chunksize=1):
        if timeout is not None:
            end_time = timeout + time.monotonic()

        fs = [self.submit(fn, *args) for args in zip(*iterables)]

        # Yield must be hidden in closure so that the futures are submitted
        # before the first iterator value is required.
        def result_iterator():
            try:
                # reverse to keep finishing order
                fs.reverse()
                while fs:
                    # Careful not to keep a reference to the popped future
                    if timeout is None:
                        yield fs.pop().result()
                    else:
                        yield fs.pop().result(end_time - time.monotonic())
            finally:
                for future in fs:
                    future.cancel()

        return result_iterator()
```

先调用`fs.reverse`转置列表，然后不断调用pop取最后一个的result。

### 死锁

```python
import time
from concurrent.futures import ThreadPoolExecutor


def a_work():
    time.sleep(5)
    print(b.result())
    return 'a'


def b_work():
    time.sleep(5)
    print(a.result())
    return 'b'


with ThreadPoolExecutor(2) as executor:
    a = executor.submit(a_work)
    b = executor.submit(b_work)
```

这个程序永远都不会运行结束，因为线程a在等待b的结果，b在等待a的结果。其中的`time.sleep(5)`非常关键，它可以保证这两个线程都开始运行了。

去掉它之后，可以发现程序居然可以运行结束了，但是却没有任何的输出。这其实就是第二个常见错误，只有调用在`future`对象上调用`result()`函数之后，异常才可以被捕获。

### 异常


```python
from concurrent.futures import ThreadPoolExecutor


def a_work():
    print(after_b)
    print(b.result())
    return 'a'


def b_work():
    print(a.result())
    return 'b'


after_b = False
with ThreadPoolExecutor(2) as executor:
    a = executor.submit(a_work)
    b = executor.submit(b_work)
    after_b = True
    print(a.result(), b.result())
```

运行上面的代码，可以发现程序输出了一个`False`，和一个异常：`NameError: name 'b' is not defined`，这是因为a线程已经开始运行了，但是b还没有开始，即，submit函数还没有返回，那么这个时候after_b的值仍然是False，而且变量b还是未定义的。

我们之前的代码，之所以没有出现异常，就是因为没有调用result函数。下面的代码可以更直观的说明问题。

```python
from concurrent.futures import ThreadPoolExecutor


def raise_exception():
    raise ValueError('123')


with ThreadPoolExecutor(1) as executor:
    a = executor.submit(raise_exception)
    # a.result()
```

运行这段代码，不会有任何输出。把`a.result()`这一行的注释取消，就可以看到异常：`ValueError: 123`。

## ProcessPoolExecutor

和`ThreadPoolExecutor`的用法基本上类似，不同之处在于，只有可以被pickle库序列化的对象才可以被执行和返回。因为需要在不同的进程之间传递消息。这个类使用了multiprocessing库。

## 为什么要使用进程池和线程池？

在我的理解看来，线程和进程的创建是需要开销的，而之前创建的线程或进程在执行完了任务之后，可以先不销毁，继续用来执行以后的任务。还有一个优势就是方便对这些创建的进程和线程进行统一的管理。

## 参考资料

官方文档：<https://docs.python.org/3/library/concurrent.futures.html>