+++
title = "Pyenv Guide on Osx"
date = 2020-04-01T11:29:34+08:00
images = []
tags = ["pyenv"]
categories = ["tool"]
draft = false
+++

> 好久不用。。发现都忘记怎么用了。赶紧写个笔记

#  Mac OSX Python开发环境配置之 pyenv

首先 打开terminal.app 或者 iterm2

`$ brew install pyenv`

然后

 `$ pyenv versions`

 检查pyenv安装完成与否

接着安装pyenv-virtualenv 插件

```shell
$ brew install pyenv-virtualenv
```

添加这几句到 .bashrc文件里

```bash
export PATH="~/.pyenv/bin:$PATH"
eval "$(pyenv init -)"
eval "$(pyenv virtualenv-init -)"
```

然后执行

```shell
$ source ~./bashrc # 如果是zsh就是~./zshrc 让当前环境变量生效
```

用法：

```shell
$ pyenv install 2.7.16
$ pyenv versions
  system
  2.7.16
  2.7.16/envs/fire_env
  3.6.8
  3.7.2
* 3.7.4 # 这个表示当前环境的Python解释器版本
$ pyenv virtualenv 2.7.16 virtual_27
```

这个时候你使用pyenv versions 你就会看到virtual_27

然后找个目录创建个目录叫

```shell
$ mkdir virtual_env
$ cd virtual_env
$ pyenv local virtual_27
(virtual_27)$ pyenv versions
system
  2.7.16
  2.7.16/envs/virtual_27
  3.6.8
  3.7.2
  3.7.4
  3.7.4/envs/374camp
  374camp
* virtual_27 (set by /Users/jay/Workspace/virtual_env/virtual_27/.python-version)
```

然后你就会发现每次，进入virtual_env 就会自动激活对应的虚拟环境了,美元符号前面的会显示你当前的虚拟环境。

notes：

   如果你遇到了这样的报告：

​     ` xxx  export PYENV_VIRTUALENV_DISABLE_PROMPT=1 xxxx`

​    那么复制`export PYENV_VIRTUALENV_DISABLE_PROMPT=1` 到命令行下面执行就好了。

使用这两个工具的好处是，可以让你的电脑上存在N个不同版本的Python，并且可以进入目录切换。

