---
title: dgraph数据库简单总结
date: 2019-04-09 11:37:09
tags:
  - dgraph
  - 数据库
---

对可扩展的，分布式的，低延迟图形数据库dgraph的用法的简单总结。

<!-- more -->

## 安装dgraph

在linux系统上，可以这样安装：
```bash
curl https://get.dgraph.io -sSf | bash
```

如果要使用docker或者从其他系统上，可以参考 <https://docs.dgraph.io/get-started/#step-1-install-dgraph>

## 启动dgraph

开3个terminal，依次启动下面的3个程序

```bash
dgraph zero
```

```bash
dgraph alpha --lru_mb 2048 --zero localhost:5080
```

```bash
dgraph-ratel
```

之后，访问`http://localhost:8000/`就可以看到一个图形化的界面。

## 端口的使用

| Dgraph Node Type | gRPC-internal | gRPC-external | HTTP-external |
| :--------------- | :------------ | :------------ | :------------ |
| zero             | –Not Used–    | 5080          | 6080          |
| alpha            | 7080          | 9080          | 8080          |
| ratel            | –Not Used–    | –Not Used–    | 8000          |

## 加载数据

若要在运行中实时加载数据和schema文件，可以使用如下的命令。

```bash
dgraph live -r g01.rdf.gz -s g01.schema -d localhost:9080 -z localhost:5080
```

详见：<https://docs.dgraph.io/deploy/#fast-data-loading>

## 插入和查询数据

有很多官方的和非官方的客户端，甚至可以使用HTTP请求来做到。

详见：<https://docs.dgraph.io/clients/>

## 导出数据

```bash
curl localhost:8080/admin/export
```

在服务器本地，向`localhost:8080/admin/export`发出`GET`请求就行了，可以导出到`~/export`文件夹下。

其它的用法，都可以查阅官方文档：<https://docs.dgraph.io/>。

