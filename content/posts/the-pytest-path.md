+++
title = "The Pytest Discovery"
date = 2020-05-08T07:38:45+08:00
images = []
tags = ["purest"]
categories = ["python"]
draft = false

+++

> 对于pytest，我一直是使用，看文档也是只看fixture那部分的。昨天跟公司的vp争论一个问题，vp觉得tests目录下不该有 `__init__.py`文件，这样子对于unite test来说语义是不对的。要使用`setup.py develop`将包装载后测试，这样就不会出现我引入`__init__.py`的初衷，为了找到找不到要测试的包。

首先，我没有认真研究为啥找不到包，但是加了`__init__.py`在tests目录就能够正常pytest命令跑完测试。也就是说我是不知道原因的情况下，误打误撞利用的pytest的机制。

为了搞清这个问题我认真阅读了[该章](https://docs.pytest.org/en/latest/goodpractices.html#tests-outside-application-code) 发现了如下的文字:

>```
>setup.py
>mypkg/
>    ...
>tests/
>    __init__.py
>    foo/
>        __init__.py
>        test_view.py
>    bar/
>        __init__.py
>        test_view.py
>```
>
>But now this introduces a subtle problem: in order to load the test modules from the `tests` directory, pytest prepends the root of the repository to `sys.path`, which adds the side-effect that now `mypkg` is also importable.This is problematic if you are using a tool like [tox](https://docs.pytest.org/en/latest/goodpractices.html#tox) to test your package in a virtual environment, because you want to test the *installed* version of your package, not the local code from the repository.

解释下，意思是，如果你通过添加`__init__.py`去改变改变测试目录，使其变为一个测试模块，那么pytest会将根目录添加到sys.path中，使得同级的python包mypkg也可以导入了。但是对于使用tox来多环境测试来说，这是有问题的。

所以mypkg也能导入这就是我能顺利运行pytest的原因，但是这种情况下，对tox使用不利，虽然公司的代码限定了python版本不需要使用tox。

[laike9m](https://laike9m.com/) 说conftest.py文件的机制能达到相同的效果，而且不需要`__init__.py` 。

找到文档描述在[这](https://docs.pytest.org/en/latest/pythonpath.html?highlight=conftest#pytest-import-mechanisms-and-sys-path-pythonpath)

但是比较奇怪，需要人为加`conftest.py` 文件在待测试包的同级别目录

```
setup.py
mypkg/
    ...
conftest.py
tests/
    foo/
        __init__.py
        test_view1.py
    bar/
        __init__.py
        test_view2.py
```

这样子也行。。

无论哪种做法都是利用了pytest，而不是将包安装进site-packages