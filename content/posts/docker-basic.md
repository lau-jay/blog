+++
title = "Docker基本使用"
date = 2016-07-12T09:13:45+08:00
images = []
tags = ["docker"]
categories = ["tool"]
draft = false
+++
## docker使用笔记
随用随记，并不成系统，主要是方便自己忘记的时候查阅。

### 如何进入运行起来的docker container
如果想要进去运行起来的container里看看日志什么的，那么使用docker 1.3
之后添加的exec

    docker exec -i -t CONTAINER_ID bash

-i 其实是STDIN，也就是使用标准输入，这样你写的指令就能在容器里执行。
-t 其实是tty的意思，就是使用终端，这样可以有个终端可以随时输入命令操作。

### 查看log

    docker log -f CONTAINTER_ID

### `.`,`./`,以及绝对路径
写习惯了docker-compose.yaml就会出现一个问题: -v 给的路径写的时候
常常会直接用相对路径`.`或者`./directory`，这在用yaml作为配置，用docker-compose编排叫起来
docker组的时候是没有问题的, 解析yaml的时候会补全的。

但是直接用docker命令叫起单个container的时候那个值可是直接就给docker的，
并不会解析补全路径，所以往往会报错。

所以之后养成的习惯就是直接用`$(pwd)`这样可以避免出错，有时候忘了这茬还是会
浪费点时间去看文档的。

### link db error

 习惯使用docker开发后，配置新项目的开发环境，经常会遇到的一个问题是数据库不存在。但是明明数据库那个docker，已经up了。
 其实原因很简单，docker link不可以`refer to localhost`。而很多框架default的数据库的URI是localhost。
 在配置中将localhost改为DB那个container的名字就好了。

### 删除退出的容器

`docker rm `docker ps -a |awk '{print $1}' | grep [0-9a-z]`

### 删除none的镜像

`docker rmi $(docker images -f 'dangling=true' -q)`

## entrypoint
今天在重新部署pyspider, 自己写了写代码利用框架，抽了个api出来，然而怎么放进docker里成了一个问题。
一开始是直接mount一个文件夹进docker，然后进入docker，将代码复制进要的目录。准备重启进程的时候尴尬的事情
发生了，docker-compose up起来的组件进程的pid是1，根本没办法kill掉再重启。

最后找到了docker-compose.yaml里command这一项，看完之后发现，都是参数。想了想去翻pyspider的Dockerfile，
于是找了作者是怎么跑的:

    ENTRYPOINT ['pyspider']

找到了怎么跑的接下来就很简单了，首先写个shell，把原本作者写的命令以及参数都写进去，在这之上加入自己
拷贝代码的shell语句。然后再在docker-compose.yaml里加上这么几行

    entrypoint:
      - bash
      - /code/replace.sh


然后再加个volume 把shell文件挂到docker的code目录就搞定了

