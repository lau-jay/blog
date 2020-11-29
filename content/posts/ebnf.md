+++
title = "EBNF-Lark Tutorial"
date = 2020-11-29T13:24:05+08:00
images = []
tags = ["translation"]
categories = []
draft = true



## Part 1 语法

Lark 接受的语法格式叫做 [EBNF](https://www.wikiwand.com/en/Extended_Backus–Naur_form). 基本的格式看起来像这样:

```ebnf
rule_name : list of rules and TERMINALS to match
          | another possible list of items
          | etc.
TERNINAL: "some text to match"
```

（TERNINAL是字符串或正则表达式）

解析器将尝试通过依次匹配其规则项（右侧）来匹配每个规则（左侧），并尝试每个替代方案（实际上，解析器是可预测的，因此我们不必尝试每个替代方案）。

如何构造这些规则不在本教程的讨论范围之内，但通常足以满足人们的直觉。

在JSON的例子中，结构很简单：JSON文档可以是列表，字典或字符串或数字等。

字典和列表是递归的， 可以包含其他JSON字符串或值。

让我们以EBNF形式编写此结构：

```ebnf
value: dict
     | list
     | STRING
     | NUMBER
     | "true" | "false" | "null"

list : "[" [value ("," value)*] "]"
dict : "{" [pair ("," pair)*] "}"
pair : STRING ":" value
```

简单的解释下语法：

* 括号让我们将规则分组在一起。

*  规则*表示任何金额。 这意味着该规则的零个或多个实例。
* [rule] 表示*可选*。 这意味着该规则的零个或一个实例

Lark also supports the rule+ operator, meaning one or more instances. It also supports the rule? operator which is another way to say *optional*.

Of course, we still haven’t defined “STRING” and “NUMBER”. Luckily, both these literals are already defined in Lark’s common library:

```
%import common.ESCAPED_STRING   -> STRING
%import common.SIGNED_NUMBER    -> NUMBER
```

The arrow (->) renames the terminals. But that only adds obscurity in this case, so going forward we’ll just use their original names.

We’ll also take care of the white-space, which is part of the text.

```
%import common.WS
%ignore WS
```

We tell our parser to ignore whitespace. Otherwise, we’d have to fill our grammar with WS terminals.

By the way, if you’re curious what these terminals signify, they are roughly equivalent to this:

```
NUMBER : /-?\d+(\.\d+)?([eE][+-]?\d+)?/
STRING : /".*?(?<!\\)"/
%ignore /[ \t\n\f\r]+/
```

Lark will accept this, if you really want to complicate your life :)

You can find the original definitions in [common.lark](https://github.com/lark-parser/lark/blob/master/lark/grammars/common.lark). They don’t strictly adhere to [json.org](https://json.org/) - but our purpose here is to accept json, not validate it.

Notice that terminals are written in UPPER-CASE, while rules are written in lower-case. I’ll touch more on the differences between rules and terminals later.