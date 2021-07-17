+++
title = "Celery踩坑记"
date = 2021-05-06T20:02:10+08:00
images = []
tags = ['celery']
categories = ['python']
draft = true

+++

背景是，Django+Celery， 但是Broker选的Redis，我也不知道为啥是Redis反正就是Redis了。

Celery出了名的坑，但是其实没啥特别好的代替就只能用了，内存泄漏这些常见问题就不记录了。反正定时重启worker大法好。

## Celery task 重复执行

​        场景是，会有一组，延迟任务，任务中凡是小于一个小时的delay的都没事，超过的都有问题。开始初步排查认为是visibility_timeout用了默认，因为延迟了1个小时没执行就没有ack，超时重新下发的问题，于是修改了该参数，改为48小时，因为有部分会延迟很长。

​       但是并没有解决该问题，之后又复现了。这次将对应的整套的log全从k8s里捞出来，对比时间段，发现是重复生产了。一摸一样的task_id生产了好几次。eta都是一样，但是任务receive的时间差在毫秒级。经过搜索结论是，用redis就是会产生这种问题。换成rabbitmq就好了。问题是不想增加中间件了。最后只能让组里的小姐姐撸一个redis锁搞定。因为反正eta在差不多时间，5分钟的锁足够了。

## Error 104 while writing to socket

​        场景是，测试服，Docker跑的Redis长时间运行之后。任务不执行，上去一看疯狂报这种错。直接贴Google里查了下，官方库的issue里发现了解决方案[github/celery](https://github.com/celery/celery/issues/4867)， 原因是Redis的缓冲区配置的问题。那么直接修改即可，我目前的做法是启动的时候直接将其全设置为0，因为只是开发环境。更具体的需要查看Redis的文档，以下`slave`需要修改为`replica`，因为政治因素，高版本的Redis不再使用slave这一词汇了。

 ```shell
 client-output-buffer-limit normal 0 0 0
 client-output-buffer-limit slave 0 0 0
 client-output-buffer-limit pubsub 0 0 0
 ```



