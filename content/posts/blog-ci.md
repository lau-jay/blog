+++
title = "Blog Ci"
date = 2017-07-15T13:40:17+08:00
images = []
tags = ["ci"]
categories = ["tool"]
draft = false
+++

比较早之前，我的博客用的是jeklly, 当时各种换插件和配置都很麻烦, 后来果断切到了hexo。
当时想要用CI来做博客的自动生成，也省得每次换电脑都要重新安装hexo，可惜当时搞岔了travis的配置，导致次生成静态页面后push到github的候就挂了。
于是一放就是几个月，这两天捡起来重新做，终于发现问题出在哪里了，于是记录下。

第一个问题是`_config.yml`
```yaml
deploy:
  type: git
  repo: git@github.com:pyclear/pyclear.github.io.git
  branch: master
```
这里需要用ssh方式而不是http方式。

第二个是我查了好久的，一个travis的安全措施导致，就算执行成功了还不会停下而是超时失败。
解决这个问题需要在.travis.yml里ssh-add那这么写

```ymal
- eval $(ssh-agent -s)
- ssh-agent -k
```
ssh-agent后需要执行带k参数的，把ssh-agent进程杀掉不然travis由于有进程在跑，会没有输出，就算执成功了状态依旧会显示执行失败的。
