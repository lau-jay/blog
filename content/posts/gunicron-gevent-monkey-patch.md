+++
title = "Gunicron Gevent Monkey Patch"
date = 2021-10-14T14:12:49+08:00
images = []
tags = ["gunicron", "gevent",  "Django"]
categories = ["python"]
draft = true

+++

## 背景

新开了一个项目，Django+Gunicorn,  Gunicorn的worker_class 采用`gevent`, 部署好测试后， 底层有个库使用requests请求第三方服务，一请求就`RecursionError: maximum recursion depth exceeded while calling a Python object` 出现这个，然后去看整个日志，会有如下警告:

```
MonkeyPatchWarning: Monkey-patching ssl after ssl has already been imported may lead to errors, including RecursionError on Python 3.6. It may also silently lead to incorrect behaviour on Python 3.7. Please monkey-patch earlier. See https://github.com/gevent/gevent/issues/1016.
```

## 解决方案

这个问题其实很好解决，issue的链接都给出来了，照着解决就好了,  在wsgi文件内， get_wsig_application之前添加如下代码即可：

 ```
 import gevent
 from gevent import monkey
 monkey.patch_all()
 ```

如果是在本地，需要在manage.py内也添加
