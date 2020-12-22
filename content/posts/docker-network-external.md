+++
title = "Docker Network External"
date = 2020-08-27T10:41:30+08:00
images = []
tags = ["docker"]
categories = ["tool"]
draft = false

+++

> 在使用docker的过程中，自建开发环境使用docker-compose是非常正常的需求，正常情况下多个容器都是通过一个docker-compose.yml唤起的，并且可以通过services名直接连接而不需要知道依赖的容器的ip地址。然而，如果遇到容器并非定义在同一个yml中的时候，连接容器就会比较麻烦。我见过直接获取对方容器ip来使用的，但是每次都得获取容器ip地址。

[参考自](https://stackoverflow.com/questions/39067295/docker-compose-external-container)

在不使用docker-compose的时候可以使用--link参数如下：

```shell
docker run --rm --name rds -d redis # 启动redis
docker run --rm --name app --link rds -d task # 启动一个应用并连接到rds
```

如果使用docker-compose就简单了:

```yaml
version: "3"
services:
    app:
       images: task
       depends_on:
         - reds
    res:
       images: redis
```

现在要解决不是一个docker-compose.yml或者是与docker run起来的容器链接的问题。

有几种解决方案，我只推荐使用以下这种方式：

首先是： app.yml

```yaml
version: "3"
services:
    app:
       images: task
       container_name: app
       networks:
         - app_net
       
networks:
  app_net:
    external: true
```

接着是: rds.yml

```yaml
version: "3"
services:
    rds:
       images: redis
       container_name: rds
       networks:
         - app_net
       
networks:
  app_net:
    external: true
```

使用的时候:

 deploy.sh :

```shell
#!/bin/bash
docker network create app_net
docker-compose -f ./deploy/rds.yml up -d
sleep 1
docker-compose -f ./deploy/app.yml up -d
```

接着可以测试下:

```shell
docker exec -it app ping rds
```

正常情况下会得到ping的结果，是通的。

如果这时候有一个已经存在的容器db，要使用的话可以:

```shell
docker network connect app_net db
```

这样三个容器都可以互通了。