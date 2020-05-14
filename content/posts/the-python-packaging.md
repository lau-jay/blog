+++
title = "The Python Packaging"
date = 2020-05-14T08:39:47+08:00
images = []
tags = ["tool", "packaging"]
categories = ["python"]
draft = false

+++

## 名字

Python的模块或者包名应该遵循下面几条:

* 总是小写
* pypi上唯一
* 不要用中划线分隔，下划线或者干脆不要分隔单词

## 最小结构

```
jay_hello/
  jay_hello/
     __init__.py
  setup.py
```

假设，`__init__.py`文件里就一个hello_world

```python
def hello_world():
  print("hello world!")
```



setup.py文件里应该是以下内容:

```python
from setuptools import setup
setup(name="jay_hello",
     version="0.0.1",
     description="just say hello",
     url="https://github.com/lau-jay/jay_hello",
     author="Lau Jay",
     author_email="example@gamil.com",
     license='MIT',
     packages=["jay_hello"],
     zip_safe=False)
```

现在可以通过下面这种方式安装

```shell
pip install . 
```

本地开发时建议使用下面这种，原因是其使用了软链接，你对代码文件的任何更改不再需要任何操作被使用

```shell
pip install -e .
```

packaging.python.org上的原文是: `in such a way that the project appears to be installed, but yet is still editable from the src tree`

那么现在就可以在repl里导入使用了

```shell
python setup.py develop 
```

这个是`pip install -e .`底层实际执行的方法，会在`site-packages`里创建`.egg-link` 文件去链接源码目录

## PyPI上发布

### 注册账户

略

### 注册名字，上传元数据以及创建包页面，创建分发包并上传

```shell
python setup.py register sdist upload
```

## 安装包

```shell
pip install jay_hello
```



## setup.py

setup参数解析

### 依赖

`install_requires`类型是字符串或者字符串列表，

ps: 19.0版本开始pip移除了对 `dependency_links` 的支持

### 提供命令行

`entry_points`：这个参数允许Python函数直接注册为命令行工具，这个机制非常可以方便地提供程序的命令行工具

```ptyhon
setup(
   ...
   entry_points = {
       "console_scripts": ["jayhello= jay_hello:hello_world"],
   }
)
```



`scripts` : 安装包的时候，setuptools会拷贝脚本到PATH，可以像Linux命令那样使用，这个字段指定的脚本不需要是Python脚本，任何可执行的都行, 假设增加了一个bin目录，并增加一个shell脚本

```
setup(
   ...
   scripts=["/bin/run.sh"]
)
```

### 添加非代码文件

如果是图像，数据表，文档等。为了让setuptools正确处理他们

需要添加个MANIFEST.in文件，例如：

```
include README.md
include docs/*.md
```

为了启用这些，需要如下设置参数:

```python
setup(
   ...
   include_package_data=True
)
```

更多参数请参见[stuptools文档](setuptools.readthedocs.io/en/latest/setuptools.html)

