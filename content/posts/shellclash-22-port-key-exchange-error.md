+++
title = "22端口踩坑记"
date = 2023-09-10T20:02:10+08:00
images = []
tags = ['proxy']
categories = ['free tech']
draft = false

+++
# 问题
git pull 我的 Github 私人仓库，通过ssh，命令行报错
```
kex_exchange_identification: Connection closed by remote host
Connection closed by 198.18.0.18 port 22
fatal: Could not read from remote repository.
```

# 问题原因
最开始看这个ip就觉得怪怪, 当然不记得了后来查了这个网段是用于子网的保留网段。第一个怀疑的是装了shellclash的路由器，直接换
了一个路由器使用电脑的clash用原来的机场配置就能直接拉代码下来。证明了两点:

* 第一 我的代理没坏，但是节点就不好说了
* 第二 我路由器可能至少是shellclash配置有问题

所以一开始直接改shellclash规则Direct github.com 但是吧, 他慢。。连网页访问都慢了。就去搜了下

# 解决方案
[这个回答解决了问题](https://github.com/vernesong/OpenClash/issues/1960)
这个回答有两个答案可以适用，第一种是Direct 22 这个网页访问可以，但是命令行还是慢
我就选了方法二直接在ssh config里把端口从22改成了443

```
Host github.com
Port 443
```





