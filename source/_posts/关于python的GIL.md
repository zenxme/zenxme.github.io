---
title: 关于python的GIL
date: 2019-04-30 17:28:24
tags:
  - python
---

对GIL(全局解释器锁)的理解。

<!-- more -->

## 什么是GIL

官方的[wiki](https://wiki.python.org/moin/GlobalInterpreterLock)中是这么说的：

> In CPython, the global interpreter lock, or GIL, is a mutex that protects access to Python objects, preventing multiple threads from executing Python bytecodes at once. This lock is necessary mainly because CPython's memory management is not thread-safe. (However, since the GIL exists, other features have grown to depend on the guarantees that it enforces.)

CPython，是官方的、用C语言写的python解释器，也是最被广泛使用的解释器。

GIL是全局解释器锁，为了避免多线程同时执行python字节码。这个锁之所以必要，是因为CPython的内存管理不是线程安全的。

所以我的题目也是有点问题的，不是Python的GIL，而是CPython的GIL。Python是一种语言，而CPython是对这个语言实现的一个解释器。除了CPython之外，还有很多Python的解释器，比如Jython、IronPython、PyPy等等。

## GIL的缺点

当时，Python的创造者Guido van Rossum在写Python的时候，多核的机器还不常见，使用一个全局的锁设计起来比较简单。但是随着硬件的发展，现在已经是多核处理器的天下了。而CPython还是只能使用一个处理器核心，这样很明显会无法充分利用计算机的性能。若只是I/O密集型任务，那么多线程还是可以提高处理效率的。然而对CPU密集型任务，非但无法提高处理速度，甚至会降低速度（由于线程的切换）。

## 能否去除GIL

由于多年的积累，CPython已经积累了大量的标准库和第三方库，它们都是在依赖GIL的情况下开发的。

虽然其它的Python解释器去除了GIL，但是可用的第三方库却很少。

直接在CPython上去掉GIL，而不影响现存的其它库是非常困难的。比如Guido的这篇博客：[It isn't Easy to Remove the GIL](<https://www.artima.com/weblogs/viewpost.jsp?thread=214235>)。在文中他提到之前有人实现了去掉GIL、换成细粒度的锁的版本，但是那样使得单线程的运行速度减慢了2倍。文章的最后，Guido写到：

> I want to point out one more time that the *language* doesn't require the GIL -- it's only the CPython virtual machine that has historically been unable to shed it.

再次强调了，不是Python语言需要GIL，而是CPython的历史原因无法去除GIL。

## 如何提升速度

- 对I/O密集型任务：多线程、协程
- 对CPU密集型任务：多进程、C语言扩展
- 使用其它解释器（但这种情况可能缺少第三方库，或是缺少Python语言的某些特性）