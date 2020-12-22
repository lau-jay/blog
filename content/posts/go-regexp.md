+++
title = "golang里被中括号坑的记录"
date = 2020-12-06T18:00:59+08:00
images = []
tags = ["bug","Hugo"]
categories = ["tool"]
draft = false

+++

今天众多大佬要用rss订阅本博客。。深感荣幸。。但是出现生成rss文件有错那就尴尬了。

于是到了茶馆打开电脑就开始修。定位了一会就定位到是特殊字符的问题。搜了下查到hugo模版里的replaceRE函数，于是就开始撸正则表达式。结果。。

## hugo replaceRE 函数好难咱不会用

hugo的replaceRE文档有这句话：

> Hugo uses Go’s [Regular Expression package](https://golang.org/pkg/regexp/),which is the same general syntax used by Perl, Python, and other languages but with a few minor differences for those coming from a background in PCRE.

于是咱就用上了浅薄的正则知识开始撸。很快就撸了一段。。确实也不报错了。于是很开心的提了个pr给用的hugo theme。回头自己看，哎生成的rss里描述的内容咋全是刚换的。知道自己刚写的正则有问题。于是各种尝试。最终的结果都是不起作用。

## regexp用的正则语法有问题？不，是眼神有问题

期间各种尝试。。反正时间很快就没了， 眼瞅着从午饭搞到晚饭了。反正都是不work。

心塞之余还是先去喝茶吧。。果然解决问题如果出现瓶颈就应该放松下。

喝完几泡茶，回到座位，看regexp的文档看到他的语法[来源](https://github.com/google/re2/wiki/Syntax)。

点击去看。。看到`[[:cntrl:]] | control (≡ [\x00-\x1F\x7F])`我突然灵机一动把原始的正则给复制过去。居然工作了。比对了下。。才发现人家有俩中括号。。坑死了。
