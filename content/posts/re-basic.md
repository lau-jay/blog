+++
title = "Python中的正则基础"
date = 2019-04-26T10:00:17+08:00
images = []
tags = ["re"]
categories = ["python"]
draft = false
+++

## Python中的re
### 元字符总览
```python
. ^ $ * + ? { } [ ] \ | ( )
```
### `[`和`]`
指定字符类， 含义是范围，匹配一组字符 `[a-z]`表示所有小写字母。
字符类里其他元字符不生效，在字符类里元字符被剥夺特殊性。
反字符类:
`[^5]` 匹配除了 5外任何字符 `^`必须是字符类第一个字符，不然无取反的作用。

### `\`
反斜杠后加其他字符的组合特指特殊序列，也用于转义。要取消本身含义需要`'\\'`。
`\w`匹配任何字母数字字符，note：依据正则表达式模式的不同，分为以字节类和Unicodedata模块中标记为字母的所有字符。
在编译时提供`re.ASCII`可以表示更受限制的`\w`定义。

类型 | 含义 |
---- | --- |
`\d` | 任何十进制数字；这等价于类 [0-9] |
`\D` | 匹配任何非数字字符；这等价于类 [^0-9]|
`\s` |匹配任何空白字符；这等价于类 [ \t\n\r\f\v]|
`\S` |匹配任何非空白字符；这相当于类 [^ \t\n\r\f\v]|
`\w` |匹配任何字母与数字字符；这相当于类 [a-zA-Z0-9_]|
`\W` |匹配任何非字母与数字字符；这相当于类 [^a-zA-Z0-9_]|


### Repeat
`*` 匹配前一个字符0～多次， 贪婪的
例子： a[bcd]*b ,匹配类`[bcd]`中的零或多个字母
`+` 匹配前一个字符1～多次
`?` 匹配0～1词
`{m,n}`  至少重复m次，最多n次。省略的话会m会是默认0，n为无限。

### 应用匹配
方法/属性 | 目的 |
---- | --- | ---
match()| 确定正则是否从字符串的开头匹配 |
search() | 扫描字符串，查找此正则匹配的任何位置。 |
findall()|找到正则匹配的所有子字符串，并将它们作为列表返回。|
finditer() |找到正则匹配的所有子字符串，并将它们返回为一个 [iterator](https://docs.python.org/zh-cn/3/glossary.html#term-iterator)|
