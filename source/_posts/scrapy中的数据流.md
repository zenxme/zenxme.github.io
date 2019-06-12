---
title: scrapy中的数据流
date: 2019-04-28 17:22:26
tags:
  - 爬虫
  - python
---

Python爬虫框架scrapy的整个数据流。
<!-- more -->

## 数据流

![](scrapy中的数据流/scrapy_architecture.png)

整个架构以Engine为中心，控制数据流在系统中的流动。

1. Request从用户编写的Spider发出，经过SpiderMiddleware到达Engine。
2. Engine将Request交给Scheduler，Scheduler把Request入队。
3. Scheduler从队列中取出Request交给Engine。
4. Request经DownloaderMiddleware到达Downloader，Downloader从互联网上下载数据，结束之后产生一个Response。
5. Downloader把Response经过DownloaderMIddleware发往Engine。
6. Engine把Response经过SpiderMiddleware交给Spider。
7. Spider可以产生Item或者Request，它们都会经过SpiderMiddleware到达Engine。
8. Engine把Request重复从2开始的步骤，把Item交给ItemPipeline进行处理。

## 参考

[官方文档](https://docs.scrapy.org/en/latest/topics/architecture.html)