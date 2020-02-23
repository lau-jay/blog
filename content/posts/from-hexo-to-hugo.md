+++
title = "From Hexo to Hugo"
date = 2020-02-22T12:49:17+08:00
images = []
tags = ["note"]
categories = ["tool"]
draft = false
+++

我之前使用hexo静态博客生成器，并使用Travis CI(Continuous Integration)来自动
生成并部署静态页面到GitHub Pages。

但是这有两个个问题:

    1. Node很麻烦，我经常装都会遇到问题。
    2. 不快

正好有时间，我决定尝试更换为[hugo](https://gohugo.io/), 搜索资料后开始行动。

> 这篇文章基于Claudio Jolowicz的文章[Hosting a Hugo blog on GitHub Pages with Travis CI](https://cjolowicz.github.io/posts/hosting-a-hugo-blog-on-github-pages-with-travis-ci/)

## 总览

总的来说你需要做以下几件事:

    1.  两个github库一个叫<username>.github.io, 一个叫啥都行推荐是blog
    2.  创建个新的github账号，建议叫<username>-blog-bot
    3.  并把<username>-blog-bot设置为<username>.github.io这个库的协作者
    4.  设置Trvais CI，以便blog库有变更会自动生成静态页面并部署

## 先将两个github仓库创建好
略

## 接着创建好新的GitHub账号
这里要说下，由于GitHub帐户必须具有唯一的电子邮件地址，
因此你需要为机器人帐户使用单独的电子邮件地址。
在这种情况下，一种有用的技术是子地址寻址（也称为plus addressing）：

    附加+blog-bot到电子邮件地址的@标志前的部分，那么发送到该地址的邮件将传递到+号前的收件箱中。

设置好了后到username.github.io的setting里加为协作者

## 安装hugo
mac OSX `brew install hugo`
其他参见[安装Hugo](https://gohugo.io/getting-started/installing/)

## 设置博客库
在你的电脑上 `hugo new site blog`

    cd blog

    git init
    git add .
    git commit -m "Initial commit"

然后 `git remote add https://github.com/username/blog.git`
接着

    git submodule add \
        https://github.com/budparr/gohugo-theme-ananke.git \
        themes/ananke
    git add .
    git commit -m "Add submodule themes/ananke"

这里我简单选择hugo入门里推荐的这个主题, 你可以自己换个。
最后 `git psuh -u origin master`

将them变成你的子模块后，就可以跟踪最新的了。

## 配置站点
配置在config.toml文件，
主要的几个配置是
```toml
    baseURL = "https://username.github.io/"
    languageCode = "zh-CN"
    title = "我的博客"
    theme = "ananke"
```
改完记得提交

## 链接部署库
还是blog目录下:

    git submodule add \
        https://github.com/username/username.github.io.git \
        public
    git commit -am "Add submodule public"

public 是最后生成的静态文件所在目录，把它跟要放的存储库关联，那么之后生成好了。
只要进这个目录`git push origin master`就会把网站部署好了

## CI
利用CI的持续部署解放自己，我们只要
  * hugo new posts/postname.md
  * git add content/posts
  * git commit -m "add post"
  * git push

这四个步骤就完成了一个博客文章从写到网站的所有过程

将更改（例如新文章）部署到博客实际需要几个步骤：
  1. 将新文章更改推送到blog存储库。也就是上面这段
  2. hugo被触发重建网站内容。
  3. 内容被推送到username.github.io存储库。
  4. 该存储库已部署到GitHub Pages。

好，接下来设置CI解决2，3两个步骤

在Travis CI(网址是travis-ci.org)上，用GitHub主账号登陆，找到blog这个存储库。
找到设置有个叫`environments`的地方,
添加一个名为的`GITHUB_AUTH_SECRET`环境变量。
使用新创建的GitHub帐户的用户名和密码将值设置为https://user:password@github.com

在本地blog库的目录下添加文件名为`.travis.yml` 的配置文件来配置Travis CI 。

blog存储库的持续集成需要执行三个任务：
  1. 将Hugo安装到CI环境中。
  2. 调用Hugo命令行工具来重建站点。
  3. 将新内容部署到username.github.io。

第三步委托给Shell脚本，这是下一部分的主题。

创建.travis.yml具有以下内容的文件：

    ---
    install:
      - curl -LO https://github.com/gohugoio/hugo/releases/download/v0.64.1/hugo_0.64.1_Linux-64bit.deb
      - sudo dpkg -i hugo_0.64.1_Linux-64bit.deb

    script:
      - hugo

    deploy:
      - provider: script
        script: bash deploy.sh
        skip_cleanup: true
        on:
          branch: master

## 添加部署脚本
在blog存储库中创建脚本，记得把下面的username替换为你的GitHub用户名：

    #!/bin/bash

    echo -e "\033[0;32mDeploying updates to GitHub...\033[0m"

    cd public

    if [ -n "$GITHUB_AUTH_SECRET" ]
    then
        touch ~/.git-credentials
        chmod 0600 ~/.git-credentials
        echo $GITHUB_AUTH_SECRET > ~/.git-credentials

        git config credential.helper store
        git config user.email "username-blog-bot@users.noreply.github.com"
        git config user.name "username-blog-bot"
    fi

    git add .
    git commit -m "Rebuild site"
    git push --force origin HEAD:master

## 最后

```bash
    git add .travis.yml deploy.sh
    git commit -am "CI: Build and push to username.github.io"
    git push
```
这样你访问https://username.github.io
就能看到你的博客了

嗯，至于怎么加文章，自己去看hugo入门吧
