---
title: 解决ubuntu 18.04下guake3.0.5输入exit会导致tab卡死的问题
date: 2019-06-13 04:01:59
tags:
    - ubuntu
    - bug
---

在我转到ubuntu16转到18的时候，遇到的guake的一个bug。
<!-- more -->

具体表现就是，在输入`exit`退出某个tab的时候，会导致这个tab卡死。

在google了一番之后，得到了2种解决方法，这两种方法都可以解决。

1. 执行`sudo apt install --reinstall libutempter0`
2. 按照这个[commit](https://github.com/Guake/guake/commit/f8699b4be6c058fd58a33a1d783cd404e9076b0e)的做法，找到guake的目录，把`guake.app`种1435行向左移动4个空格即可。

参考链接：

- <https://bugs.launchpad.net/ubuntu/+source/guake/+bug/1760621>
- <https://github.com/Guake/guake/issues/1198>