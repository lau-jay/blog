---
title: "From Pickle to Serialization"
date: 2021-07-25T10:15:56+08:00
draft: true
---

# 从Pickle到序列化与反序列化

项目中采用了一个库，作用是Django ORM的QuerySet缓存起来，减少数据库查询，因为QuerySet的内置的cache其实只能用来在一个上下文通过变量来做到，离开同一个上下文就没办法了。所以引入了一个缓存包，以查询条件为key做缓存。

##  从Pickle的一个报错开始

在一次测试环境发布之后，访问接口意外报了个错：

```
ValueError: unsupported pickle protocol: 5
```

但是之前并未出现过该错误，之前为了稳妥看过缓存包的实现，所以第一时间就想到了该包，顺手贴了报错去SO查了下，说是Python 3.8才会有pickle protocol 5的协议。一问果然有人本地环境是3.8.x，因为我们开发是直连的测试环境的Redis和DB，那么会有这种报错也就不奇怪了，虽然测试环境没开启缓存，但是因为有少部分默认的配置存在，所以还是存在序列化的数据的。直接将Redis的缓存清空就解决了该问题。但是这个引起了我的好奇心，去研究了下pickle protocol 5的由来以及更进一步的系统了解了序列化和反序列化

## Pickle Protocol 5

Pickle Protocol 5 来自[PEP574](https://www.python.org/dev/peps/pep-0574/) ，进入的Python版本是3.8，该PEP的基本原理的最后一句话：

> This PEP aims to make `pickle` usable in a way where large data is handled as a separate stream of zero-copy buffers, letting the application handle those buffers optimally.

说明了这个版本的协议的主要目的。
