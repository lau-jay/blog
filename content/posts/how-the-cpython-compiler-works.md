+++
title = "Python的幕后#2: CPython 编译器是如何工作的"
date = 2021-01-02T20:32:50+08:00
images = []
tags = []
categories = []
draft = true

+++

> [原文](https://tenthousandmeters.com/blog/python-behind-the-scenes-2-how-the-cpython-compiler-works/)
>
> 本文已获原文作者[Victor Skvortsov](https://tenthousandmeters.com/about/)授权

在本系列的 [第一篇](https://laujay.com/posts/how-the-cpython-vm-works/) 中我们介绍了CPython VM。我们已经了解到它可以通过执行一系列称为字节码的指令来工作。 我们还看到，Python字节码不足以完整描述一段代码的功能。 这就是为什么存在代码对象的原因。执行代码块比如模块或者函数意味着执行相应的代码对象。代码对象包含了代码块的字节码， 该代码块中使用的常量和变量以及代码块的各种属性。

通常，Python程序员不编写字节码，也不创建代码对象，而是编写普通的Python代码。 因此，CPython必须能够从源代码创建代码对象。 这项工作由CPython编译器完成。 在这一部分中，我们将探讨其工作原理。

**Note**: 在本文中，我指的是CPython 3.9。 随着CPython的发展，某些实现细节肯定会发生变化。 我将尝试跟踪重要的更改并添加更新说明。

### 什么是CPython 编译器？

我们了解了CPython编译器的职责，但是在研究其实现方式之前，让我们弄清楚为什么首先将其称为编译器。

一般而言，编译器是将一种语言的程序等价翻译成另一种语言的程序的程序。 编译器的类型很多，但是在大多数情况下，编译器是指静态编译器，它可以将高级语言的程序转换为机器代码。 CPython编译器与这种类型的编译器有共同点吗？ 为了回答这个问题，让我们看一下传统静态编译器的三阶段设计。

[图1](/img/diagram1.png)

编译器的前端将源代码转换为某种中间表示（IR）。 然后，优化器获取一个IR，对其进行优化，然后将优化的IR传递给生成机器代码的后端。 如果我们选择不是特定于任何源语言和任何目标机器的IR，那么我们将获得三阶段设计的主要好处：对于支持新语言的编译器，仅需要一个附加的前端，并且支持一个新的目标平台，只需要一个新的后端。

LLVM工具链是该模型成功的一个很好的例子。 有一些C，Rust，Swift和许多其他编程语言的前端，它们依赖LLVM提供编译器的更复杂部分。 LLVM的创建者Chris Lattner很好地概述了[其体系结构](http://aosabook.org/en/llvm.html)。

但是，CPython不需要支持多种语言和目标平台，而只需支持Python代码和CPython VM。 尽管如此，CPython编译器的实现是三阶段的。 要了解原因，我们应该更详细地研究三阶段编译器的结构。

[图2](/img/diagram2.png)

上图是经典编译器的模型。 现在将其与下图中的CPython编译器的体系结构进行比较。

[图3](/img/diagram3.png)

看起来很相似，不是吗？ 这里的重点是，以前学习过编译器的任何人都应该熟悉CPython编译器的结构。如果没有的话，一本著名的[Dragon Book](https://en.wikipedia.org/wiki/Compilers:_Principles,_Techniques,_and_Tools)是对编译器构造理论的出色介绍。 它很长，但是即使只阅读前几章，您也会从中受益。

我们进行的比较需要提前解释一些东西。 首先，从3.9版本开始，CPython默认使用一个新的解析器，该解析器立即输出AST（抽象语法树），而无需进行构建解析树的中间步骤。 因此，CPython编译器的模型被进一步简化。 其次，与静态编译器的相应阶段相比，CPython编译器的某些现成阶段做得很少，有人可能会说CPython编译器不过是前端。 我们不会采用hardcore compiler 编写者的这种观点。

### 编译器架构概述

这些图很不错，但是它们隐藏了许多细节并且可能会引起误解，因此让我们花一些时间来讨论CPython编译器的总体设计。

CPython编译器的两个主要组件是：

1. 前端和
2. 后端

前端获取Python代码并生成AST。后端获取AST并生成一个代码对象。在整个CPython源代码中，术语“解析器”和“编译器”分别用于前端和后端。这是编译器一词的另一含义。最好将其称为类似代码对象生成器的名称，但是我们会坚持使用编译器，因为它似乎不会造成太多麻烦。

解析器的工作是检查输入是否在语法上正确的Python代码。 如果不是，则解析器报告如下错误：

```python
x = y = = 12
        ^
SyntaxError: invalid syntax
```

如果输入正确，将那么解析器将根据语法规则对其进行组织。语法定义语言的语法。 形式语法的概念对于我们的讨论是如此重要，我想我们应该花点时间回顾一下它的正式定义。

根据经典定义，语法是一个包含四个项目的元组：

- Σ – 一组有限的终结符，或简单的终结符（通常用小写字母表示）
- N – 一组有限的非终结符号，或简称为非终结符（通常用大写字母表示）。
- P – 生成式, 一套生产规则。 对于上下文无关的语法（包括Python语法），生产规则只是从非终结符到任意序列的终结符和非终结符的映射，例如 A→aB。
- S – 识别符.

文法定义了一种语言，其中包含可以通过应用生成式生成的所有终结符序列。为了生成某个序列，以符号S开头，然后根据生成式将每个非终结符号递归替换为一个序列，直到整个序列由终结符组成。 使用已建立的约定惯例，列出生成式以指定语法就足够了。 例如，这是一个简单的语法，该语法生成交替的一和零的序列:

S→10S|10

当我们更详细地分析解析器时，我们将继续讨论文法。

### Abstract syntax tree

解析器的最终目标是生成AST。AST是一种树形数据结构，用作源代码的高级表示。这是标准ast模块产生的相应[AST](https://docs.python.org/3/library/ast.html)的一段代码和转储的示例

```python
x = 123
f(x)
```

```shell
$ python -m ast example1.py
Module(
   body=[
      Assign(
         targets=[
            Name(id='x', ctx=Store())],
         value=Constant(value=123)),
      Expr(
         value=Call(
            func=Name(id='f', ctx=Load()),
            args=[
               Name(id='x', ctx=Load())],
            keywords=[]))],
   type_ignores=[])
```

AST节点的类型使用[Zephyr抽象文法定义语言](https://www.cs.princeton.edu/research/techreps/TR-554-97) (ASDL) 。ASDL是一种简单的声明性语言，用于描述树状IR，即AST。这是[Parser/Python.asdl](https://github.com/python/cpython/blob/master/Parser/Python.asdl)中的Assign和Expr节点的定义：

```python
stmt = ... | Assign(expr* targets, expr value, string? type_comment) | ...
expr = ... | Call(expr func, expr* args, keyword* keywords) | ...
```

ASDL规范让我们了解了Python AST的样子。但是，解析器需要在C代码中表示AST。幸运的是，从AST节点的ASDL描述中生成C结构很容易。这就是CPython所做的，结果如下所示：

```c
struct _stmt {
    enum _stmt_kind kind;
    union {
        // ... other kinds of statements
        struct {
            asdl_seq *targets;
            expr_ty value;
            string type_comment;
        } Assign;
        // ... other kinds of statements
    } v;
    int lineno;
    int col_offset;
    int end_lineno;
    int end_col_offset;
};

struct _expr {
    enum _expr_kind kind;
    union {
        // ... other kinds of expressions
        struct {
            expr_ty func;
            asdl_seq *args;
            asdl_seq *keywords;
        } Call;
        // ... other kinds of expressions
    } v;
    // ... same as in _stmt
};
```

AST是一种易于使用的表示形式。 它告诉程序做什么，隐藏所有不必要的信息，例如缩进，标点和其他Python的句法特性。

AST表示法的主要受益者之一是编译器， which can walk an AST and emit bytecode in a relatively straightforward manner. 除了编译器之外，许多Python工具都使用AST来处理Python代码。例如，[pytest](https://github.com/pytest-dev/pytest/)在`assert`语句失败时，对AST进行更改以提供有用的信息，它本身不执行任何操作，但是如果表达式的计算结果为`False`，则会引发`AssertionError`。 另一个例子是[Bandit](https://github.com/PyCQA/bandit)，它通过分析AST在Python代码中发现常见的安全问题。 

现在，当我们稍微学习了Python AST之后，我们可以看看解析器是如何从源代码中构建它的。

### 从源代码到AST

事实上，正如我前面所说，从3.9版本开始，CPython的解析器不是一个，而是两个。新的解析器是默认使用的，也可以通过传递`-X oldparser`选项来使用旧的解析器。 但在CPython 3.10中，旧的解析器将被完全删除。这两个解析器有很大的不同。我们将重点讨论新的解析器，但在此之前，先讨论一下旧的解析器。

#### 旧的解释器

在很长一段时间里，Python的语法是由生成式文法正式定义的。在很长一段时间里，Python的语法是由生成式文法正式定义的。问题在于，生成式文法并不能直接对应于能够解析这些序列的解析算法。幸运的是，聪明的人已经能够区分出生成文法的类别，并为其建立相应的解析器。这些文法包括[上下文无关](https://en.wikipedia.org/wiki/Context-free_grammar)、[LL(k)](https://en.wikipedia.org/wiki/LL_grammar)、[LR(k)](https://en.wikipedia.org/wiki/LR_parser)、[LALR](https://en.wikipedia.org/wiki/LALR_parser) 和许多其他类型的文法。Python文法是LL(1)。它使用一种扩展的 Backus-Naur 形式 ([EBNF](https://en.wikipedia.org/wiki/Extended_Backus%E2%80%93Naur_form)) 来指定。为了了解如何使用它来描述Python的语法，请看一下while语句的规则。

```
file_input: (NEWLINE | stmt)* ENDMARKER
stmt: simple_stmt | compound_stmt
compound_stmt: ... | while_stmt | ...
while_stmt: 'while' namedexpr_test ':' suite ['else' ':' suite]
suite: simple_stmt | NEWLINE INDENT stmt+ DEDENT
...
```

CPython扩展了传统记法，具有以下特点:

* 替代分组: (a | b)
* 选择部分: [a]
* 零个或多个和一个或多个重复: a* 和 a+

我们可以看到[为什么Guido van Rossum选择使用正则表达式](https://www.blogger.com/profile/12821714508588242516)。它们允许以一种更自然（对程序员来说）的方式来表达编程语言的语法。不写A→aA|a ，我们可以直接写A→a+。这个选择是有代价的。CPython必须开发一种方法来支持扩展符号。

