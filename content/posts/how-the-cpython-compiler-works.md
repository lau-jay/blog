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

#### 旧的文法分析器

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

LL(1)文法的解析是一个已解决的问题。解决的方法是作为自上而下的解析器的[下推自动机](https://en.wikipedia.org/wiki/Pushdown_automaton)（PDA）。PDA通过使用栈模拟输入字符串的生成进行操作。为了解析一些输入，它从栈上的起始符号开始。然后，它查看输入中的第一个符号，猜测应该对起始符号应用哪条规则，并将其替换为该规则的右侧。如果栈上的栈顶符号是一个终结符，与输入中的下一个符号相匹配，PDA就会把它出栈并跳过匹配的符号。如果栈顶符号是一个非终结符，PDA会根据输入中的下一个符号，尝试猜测要替换它的规则。这个过程一直重复，直到扫描完整个输入，或者如果PDA无法将栈上的一个终结符与输入中的下一个符号相匹配。后一种情况意味着输入字符串无法被解析。

CPython由于生成式的写法，不能直接使用这种方法，所以必须开发新的方法。

为了支持扩展的符号，旧的解析器用[确定性有限自动机](https://en.wikipedia.org/wiki/Deterministic_finite_automaton)(DFA)来表示语法的每条规则，DFA以等价于正则表达式而闻名。解析器本身是一个像PDA一样的基于栈的自动机，但它不是在栈上推送符号，而是推送DFA的状态。下面是老解析器使用的关键数据结构。

```c
typedef struct {
    int              s_state;       /* State in current DFA */
    const dfa       *s_dfa;         /* Current DFA */
    struct _node    *s_parent;      /* Where to add next node */
} stackentry;

typedef struct {
    stackentry      *s_top;         /* Top entry */
    stackentry       s_base[MAXSTACK];/* Array of stack entries */
                                    /* NB The stack grows down */
} stack;

typedef struct {
    stack           p_stack;        /* Stack of parser states */
    grammar         *p_grammar;     /* Grammar to use */
                                    // basically, a collection of DFAs
    node            *p_tree;        /* Top of parse tree */
    // ...
} parser_state;
```

还有[Parser/parser.c](https://github.com/python/cpython/blob/3.9/Parser/parser.c)中的注释，总结了这个方法:

>一个解析规则用一个确定性有限状态自动机（DFA）来表示。DFA中的一个节点代表解析器的一个状态；一个弧线代表一个过渡。状态迁移要么用终结符标注，要么用非终结符标注。当解析器决定跟随一个标有非终结符的弧线时，它就会被递归调用，以代表该解析规则的DFA作为初始状态；当该DFA接受时，调用它的解析器就会继续。递归调用的解析器构建的解析树作为子树插入到当前的解析树中。

解析器在解析输入时，会建立一棵解析树，也称为具体语法树（CST）。与AST相反，解析树直接对应于推导输入时应用的规则。解析树中的所有节点都使用同一个节点结构来表示。

```c
typedef struct _node {
    short               n_type;
    char                *n_str;
    int                 n_lineno;
    int                 n_col_offset;
    int                 n_nchildren;
    struct _node        *n_child;
    int                 n_end_lineno;
    int                 n_end_col_offset;
} node;
```

然而，一个解析树并不是编译器所期待的。它必须被转换为AST。这项工作是在[Python/ast.c](https://github.com/python/cpython/blob/3.9/Python/ast.c)中完成的，算法是递归地遍历一棵解析树，并将其节点转换成AST节点。几乎没有人觉得这近6000行代码很惊艳。

#### 词法分析器

从语法的角度来看，Python不是一门简单的语言。不过，Python 文法看起来很简单，包括注释在内，大约可以只有 200 行。这是因为文法的符号是标记而不是单个字符。一个符号由类型来表示，如NUMBER、NAME、NEWLINE、值和在源代码中的位置。CPython区分了63种符号类型，这些类型都列在[文法/符号](https://github.com/python/cpython/blob/3.9/Grammar/Tokens)中。我们可以使用标准的[tokenize](https://docs.python.org/3/library/tokenize.html)模块来查看一个词法分析后的程序的样子。

```python
def x_plus(x):
    if x >= 0:
        return x
    return 0
```

```shell
$ python -m tokenize example2.py 
0,0-0,0:            ENCODING       'utf-8'        
1,0-1,3:            NAME           'def'          
1,4-1,10:           NAME           'x_plus'       
1,10-1,11:          OP             '('            
1,11-1,12:          NAME           'x'            
1,12-1,13:          OP             ')'            
1,13-1,14:          OP             ':'            
1,14-1,15:          NEWLINE        '\n'           
2,0-2,4:            INDENT         '    '         
2,4-2,6:            NAME           'if'           
2,7-2,8:            NAME           'x'            
2,9-2,11:           OP             '>='           
2,12-2,13:          NUMBER         '0'            
2,13-2,14:          OP             ':'            
2,14-2,15:          NEWLINE        '\n'           
3,0-3,8:            INDENT         '        '     
3,8-3,14:           NAME           'return'       
3,15-3,16:          NAME           'x'            
3,16-3,17:          NEWLINE        '\n'           
4,4-4,4:            DEDENT         ''             
4,4-4,10:           NAME           'return'       
4,11-4,12:          NUMBER         '0'            
4,12-4,13:          NEWLINE        '\n'           
5,0-5,0:            DEDENT         ''             
5,0-5,0:            ENDMARKER      ''     
```

这是该程序在文法解析器中的样子。当文法解析器需要一个符号时，它就会向词法分析器请求一个符号。词法分析器每次从缓冲区中读取一个字符，并尝试将看到的前缀与某种类型的符号进行匹配。词法分析器如何处理不同的编码？它依赖于`io`模块。首先，词法分析器检测编码。如果没有指定编码，它默认为UTF-8。然后，词法分析器用 C 调用打开一个文件，相当于 Python 的 `open(fd, mode='r', encoding=enc)`，并通过调用`readline()`函数读取文件内容。这个函数返回一个unicode字符串。词法分析器读取的字符只是该字符串UTF-8表示的字节（或EOF）。

我们可以直接在文法中定义一个数字或一个名称是什么，尽管这会变得更加复杂。我们不能做的是在文法中表达缩进的意义，而不使其成为[上下文敏感的](https://en.wikipedia.org/wiki/Context-sensitive_grammar)，因此，缩进不适合解析。词法分析器通过提供 `INDENT `和 `DEDENT `符号，使解析器的工作变得更加容易。它们的意思就是像 C 语言中大括号的意思。词法分析器有足够的能力来处理缩进，因为它有状态。当前的缩进级别被保存在栈的顶部。当级别增加时，它的缩进会被推送到栈上。如果级别降低，所有更高的级别都会从堆栈中弹出。

旧的文法分析器是CPython代码库中的一个不小的部分。文法的DFA是自动生成的，但文法分析器的其他部分是手工编写的。这与新的文法分析器形成鲜明对比，新的文法分析器似乎是解决Python代码解析问题的一个更优雅的方案。

#### 新的文法分析器

新的解析器带有新的文法。该文法法为[解析表达语法](https://en.wikipedia.org/wiki/Parsing_expression_grammar)（PEG）。要明白的是，PEG不仅仅是一类文法。 这是另一种定义文法的方式。PEGs是[由Bryan Ford在2004年提出的](https://pdos.csail.mit.edu/~baford/packrat/popl04/)，作为描述一种编程语言并根据描述生成解析器的工具 。PEG与传统的形式化文法不同的是，它的规则将非终结符映射到解析表达式上，而不仅仅是符号序列。 这与CPython的理念是一致的。 解析表达式是归纳性定义的。 如果e、e1和e2是解析表达式,  那就意味着它是:

1. 空字符串
2. 任何终结符
3. 任何非终结符
4. e1e2, 一个序列
5. e1/e2, 优先选择
6. e∗, 零次或多次重复
7. !e, 非谓词

PEGs是分析文法，这意味着它们不仅可以生成语言，还可以分析它们。Ford将解析表达式e识别输入x的含义形式化，基本上，任何用某个解析表达式识别输入的尝试，要么成功，要么失败,  消费掉输入x或者不消费。例如，将解析表达式a应用于输入ab的结果是成功，并消费掉a。

这种形式化允许将任何PEG转换为[递归下降解析器](https://en.wikipedia.org/wiki/Recursive_descent_parser) 。递归下降解析器将文法的每个非终结符与解析函数相关联。 对于PEG来说，解析函数的功能是相应解析表达式的实现。 如果解析表达式包含非终结符，则将其递归调用其解析函数。

一个非终结符可能有多个产生规则。递归下降解析器必须决定哪一个是用来推导输入的。如果一个文法是LL(k)，一个解析器可以查看输入中的下一个k个字符，并预测正确的规则。这样的解析器称为预测性解析器。如果无法预测，则采用回溯法。采用回溯法的解析器会尝试一条规则，如果失败，则回溯并尝试另一条规则。这正是PEG中优先选择操作符的作用。所以，PEG解析器是一个带回溯的递归下降解析器。

回溯方法功能强大，但计算成本很高。 考虑一个简单的例子。 我们将表达式AB/A 应用于在A上成功但在B上失败的输入。根据对优先选择运算符的解释，解析器首先尝试识别A，成功，然后尝试识别B。失败后再尝试识别A。由于这样的冗余计算，解析时间可能是输入大小的指数。为了解决这个问题, [Ford suggested](https://bford.info/pub/lang/packrat-icfp02/) 使用记忆技术，即缓存函数调用的结果。使用此技术，可以保证解析器（称为packrat解析器）可以在线性时间内工作，但要消耗更多的内存。 这就是CPython的新解析器所做的。 这是一个packrat解析器！

无论新的解析器有多好，都必须给出替换旧解析器的理由。这就是PEPs的作用。 [PEP 617 -- New PEG parser for CPython](https://www.python.org/dev/peps/pep-0617/) 给出了新旧解析器的背景，并解释了过渡背后的原因。简而言之，新的解析器取消了对语法的LL(1)限制，应该更容易维护。Guido van Rossum 写了 [一个系列关于PEG解析的优秀文章](https://medium.com/@gvanrossum_83706/peg-parsing-series-de5d41b2ed60), 在这写文章里他详细介绍了如何实现一个简单的PEG解析器。接下来，我们将看看它的CPython实现。

您可能会惊讶地发现，[新的文法文件](https://github.com/python/cpython/blob/3.9/Grammar/python.gram)比旧的大了三倍多。这是因为新的文法不仅仅是一个文法，而是一个[语法导向翻译方案](https://en.wikipedia.org/wiki/Syntax-directed_translation)(SDTS)。SDTS是一个在规则上附加了动作的文法。一个动作是一段代码。当解析器将相应的规则应用于输入并成功时，它就会执行一个动作。CPython在解析时使用action来构建AST。要想知道是怎么做的，我们来看看新的语法是什么样子的。我们已经看到了旧语法的while语句的规则，所以这里是它们的新样子。

```
file[mod_ty]: a=[statements] ENDMARKER { _PyPegen_make_module(p, a) }
statements[asdl_seq*]: a=statement+ { _PyPegen_seq_flatten(p, a) }
statement[asdl_seq*]: a=compound_stmt { _PyPegen_singleton_seq(p, a) } | simple_stmt
compound_stmt[stmt_ty]:
    | ...
    | &'while' while_stmt
while_stmt[stmt_ty]:
    | 'while' a=named_expression ':' b=block c=[else_block] { _Py_While(a, b, c, EXTRA) }
...
```

每个规则都以非终结符的名称开头。 解析函数返回结果的C类型。 右侧是解析表达式。 花括号中的代码表示一个动作。 动作是返回AST节点或其字段的简单函数调用。

新的解析器是[Parser/pegen / parse.c](https://github.com/python/cpython/blob/3.9/Parser/pegen/parse.c)。它由解析器生成器自动生成。解析器生成器是用Python编写的。这是一个用C或Python获取语法并生成PEG解析器的程序。文法在文法文件中描述，并由`Grammar`类的实例表示。要创建这样的实例，文法文件必须有一个解析器。 [此解析器](https://github.com/python/cpython/blob/3.9/Tools/peg_generator/pegen/grammar_parser.py)也是由解析器生成器自动从[metagrammar](https://github.com/python/cpython/blob/3.9/Tools/peg_generator/pegen/grammar_parser.py)。这就是解析器生成器可以在Python中生成解析器的原因。但是解析metagrammar的是什么呢？好吧，它与语法的符号相同，因此生成的文法解析器也能够解析metagrammar。当然，文法分析器必须是自举的，第一个版本必须是手工编写的。完成后，所有解析器都可以自动生成。

与旧的解析器一样，新的解析器从词法分析器获取符号。 对于PEG解析器而言，这是特殊的，因为它可以统一标记和解析。 直到我们看到了tokenizer所做的非凡的工作，因此CPython开发人员决定使用它。

到此为止，我们结束了对解析的讨论，接下来看下AST。

### AST 优化

CPython编译器的架构图向我们展示了AST优化器以及解析器和编译器的情况。这可能过分强调了优化器的作用。AST优化器仅限于常量折叠，在CPython 3.7中才引入。在CPython 3.7之前，常量折叠是由窥孔优化器在后期完成的。然而，由于AST优化器的存在，我们可以写这样的东西:

```
n = 2 ** 32 # easier to write and to read
```

并可以预期在编译时计算出来。

一个不太明显的优化例子是将一个常量列表和一个常量集分别转换为一个元组和一个frozenset。当在` in`或 `not in `运算符的右侧使用列表或集合时，会进行这种优化。

### 从AST到代码对象

到目前为止，我们一直在研究CPython是如何从源代码中创建AST的，但正如我们在第一篇文章中看到的，CPython虚拟机对AST一无所知，只能执行一个代码对象。将AST转换为代码对象是编译器的工作。更具体地说，编译器必须返回模块的代码对象，其中包含模块的字节码以及模块中其他代码块的代码对象，如定义的函数和类。

Sometimes the best way to understand a solution to a problem is to think of one's own. Let's ponder what we would do if we were the compiler. We start with the root node of an AST that represents a module. Children of this node are statements. Let's assume that the first statement is a simple assignment like `x = 1`. It's represented by the `Assign` AST node: `Assign(targets=[Name(id='x', ctx=Store())], value=Constant(value=1))`. To convert this node to a code object we need to create one, store constant `1` in the list of constants of the code object, store the name of the variable `x` in the list of names used in the code object and emit the `LOAD_CONST` and `STORE_NAME` instructions. We could write a function to do that. But even a simple assignment can be tricky. For example, imagine that the same assignment is made inside the body of a function. If `x` is a local variable, we should emit the `STORE_FAST` instruction. If `x` is a global variable, we should emit the `STORE_GLOBAL` instruction. Finally, if `x` is referenced by a nested function, we should emit the `STORE_DEREF` instruction. The problem is to determine what the type of the variable `x` is. CPython solves this problem by building a symbol table before compiling.

#### symbol table

A symbol table contains information about code blocks and the symbols used within them. It's represented by a single `symtable` struct and a collection of `_symtable_entry` structs, one for each code block in a program. A symbol table entry contains the properties of a code block, including its name, its type (module, class or function) and a dictionary that maps the names of variables used within the block to the flags indicating their scope and usage. Here's the complete definition of the `_symtable_entry` struct:

```c
typedef struct _symtable_entry {
    PyObject_HEAD
    PyObject *ste_id;        /* int: key in ste_table->st_blocks */
    PyObject *ste_symbols;   /* dict: variable names to flags */
    PyObject *ste_name;      /* string: name of current block */
    PyObject *ste_varnames;  /* list of function parameters */
    PyObject *ste_children;  /* list of child blocks */
    PyObject *ste_directives;/* locations of global and nonlocal statements */
    _Py_block_ty ste_type;   /* module, class, or function */
    int ste_nested;      /* true if block is nested */
    unsigned ste_free : 1;        /* true if block has free variables */
    unsigned ste_child_free : 1;  /* true if a child block has free vars,
                                     including free refs to globals */
    unsigned ste_generator : 1;   /* true if namespace is a generator */
    unsigned ste_coroutine : 1;   /* true if namespace is a coroutine */
    unsigned ste_comprehension : 1; /* true if namespace is a list comprehension */
    unsigned ste_varargs : 1;     /* true if block has varargs */
    unsigned ste_varkeywords : 1; /* true if block has varkeywords */
    unsigned ste_returns_value : 1;  /* true if namespace uses return with
                                        an argument */
    unsigned ste_needs_class_closure : 1; /* for class scopes, true if a
                                             closure over __class__
                                             should be created */
    unsigned ste_comp_iter_target : 1; /* true if visiting comprehension target */
    int ste_comp_iter_expr; /* non-zero if visiting a comprehension range expression */
    int ste_lineno;          /* first line of block */
    int ste_col_offset;      /* offset of first line of block */
    int ste_opt_lineno;      /* lineno of last exec or import * */
    int ste_opt_col_offset;  /* offset of last exec or import * */
    struct symtable *ste_table;
} PySTEntryObject;
```

CPython uses the term namespace as a synonym for a code block in the context of symbol tables. So, we can say that a symbol table entry is a description of a namespace. The symbol table entries form an hierarchy of all namespaces in a program through the `ste_children` field, which is a list of child namespaces. We can explore this hierarchy using the standard [`symtable`](https://docs.python.org/3/library/symtable.html#module-symtable) module:

```python
# example3.py
def func(x):
    lc = [x+i for i in range(10)]
    return lc
>>> from symtable import symtable
>>> f = open('example3.py')
>>> st = symtable(f.read(), 'example3.py', 'exec') # module's symtable entry
>>> dir(st)
[..., 'get_children', 'get_id', 'get_identifiers', 'get_lineno', 'get_name',
 'get_symbols', 'get_type', 'has_children', 'is_nested', 'is_optimized', 'lookup']
>>> st.get_children()
[<Function SymbolTable for func in example3.py>]
>>> func_st = st.get_children()[0] # func's symtable entry
>>> func_st.get_children()
[<Function SymbolTable for listcomp in example3.py>]
>>> lc_st = func_st.get_children()[0] # list comprehension's symtable entry
>>> lc_st.get_symbols()
[<symbol '.0'>, <symbol 'i'>, <symbol 'x'>]
>>> x_sym = lc_st.get_symbols()[2]
>>> dir(x_sym)
[..., 'get_name', 'get_namespace', 'get_namespaces', 'is_annotated',
 'is_assigned', 'is_declared_global', 'is_free', 'is_global', 'is_imported',
 'is_local', 'is_namespace', 'is_nonlocal', 'is_parameter', 'is_referenced']
>>> x_sym.is_local(), x_sym.is_free()
(False, True)
```

This example shows that every code block has a corresponding symbol table entry. We've accidentally come across the strange `.0` symbol inside the namespace of the list comprehension. This namespace doesn't contain the `range` symbol, which is also strange. This is because a list comprehension is implemented as an anonymous function and `range(10)` is passed to it as an argument. This argument is referred to as `.0`. What else does CPython hide from us?

The symbol table entries are constructed in two passes. During the first pass, CPython walks the AST and creates a symbol table entry for each code block it encounters. It also collects information that can be collected on the spot, such as whether a symbol is defined or used in the block. But some information is hard to deduce during the first pass. Consider the example:

```python
def top():
    def nested():
        return x + 1
    x = 10
    ...
```

When constructing a symbol table entry for the `nested()` function, we cannot tell whether `x` is a global variable or a free variable, i.e. defined in the `top()` function, because we haven't seen an assignment yet.

CPython solves this problem by doing the second pass. At the start of the second pass it's already known where the symbols are defined and used. The missing information is filled by visiting recursively all the symbol table entries starting from the top. The symbols defined in the enclosing scope are passed down to the nested namespace, and the names of free variables in the enclosed scope are passed back.

The symbol table entries are managed using the `symtable` struct. It's used both to construct the symbol table entries and to access them during the compilation. Let's take a look at its definition:

```c
struct symtable {
    PyObject *st_filename;          /* name of file being compiled,
                                       decoded from the filesystem encoding */
    struct _symtable_entry *st_cur; /* current symbol table entry */
    struct _symtable_entry *st_top; /* symbol table entry for module */
    PyObject *st_blocks;            /* dict: map AST node addresses
                                     *       to symbol table entries */
    PyObject *st_stack;             /* list: stack of namespace info */
    PyObject *st_global;            /* borrowed ref to st_top->ste_symbols */
    int st_nblocks;                 /* number of blocks used. kept for
                                       consistency with the corresponding
                                       compiler structure */
    PyObject *st_private;           /* name of current class or NULL */
    PyFutureFeatures *st_future;    /* module's future features that affect
                                       the symbol table */
    int recursion_depth;            /* current recursion depth */
    int recursion_limit;            /* recursion limit */
};
```

The most important fields to note are `st_stack` and `st_blocks`. The `st_stack` field is a stack of symbol table entries. During the first pass of the symbol table construction, CPython pushes an entry onto the stack when it enters the corresponding code block and pops an entry from the stack when it exits the corresponding code block. The `st_blocks` field is a dictionary that the compiler uses to get a symbol table entry for a given AST node. The `st_cur` and `st_top` fields are also important but their meanings should be obvious.

To learn more about symbol tables and their construction, I highly recommend you [the articles by Eli Bendersky](https://eli.thegreenplace.net/2010/09/18/python-internals-symbol-tables-part-1).

#### basic blocks

A symbol table helps us to translate statements involving variables like `x = 1`. But a new problem arises if we try to translate a more complex control-flow statement. Consider another cryptic piece of code:

```python
if x == 0 or x > 17:
    y = True
else:
    y = False
...
```

The corresponding AST subtree has the following structure:

```c
If(
  test=BoolOp(...),
  body=[...],
  orelse=[...]
)
```

And the compiler translates it to the following bytecode:

```shell
1           0 LOAD_NAME                0 (x)
            2 LOAD_CONST               0 (0)
            4 COMPARE_OP               2 (==)
            6 POP_JUMP_IF_TRUE        16
            8 LOAD_NAME                0 (x)
           10 LOAD_CONST               1 (17)
           12 COMPARE_OP               4 (>)
           14 POP_JUMP_IF_FALSE       22

2     >>   16 LOAD_CONST               2 (True)
           18 STORE_NAME               1 (y)
           20 JUMP_FORWARD             4 (to 26)

4     >>   22 LOAD_CONST               3 (False)
           24 STORE_NAME               1 (y)
5     >>   26 ...
```

The bytecode is linear. The instructions for the `test` node should come first, and the instructions for the `body` block should come before those for the `orelse` block. The problem with the control-flow statements is that they involve jumps, and a jump is often emitted before the instruction it points to. In our example, if the first test succeeds, we would like to jump to the first `body` instruction straight away, but we don't know where it should be yet. If the second test fails, we have to jump over the `body` block to the `orelse` block, but the position of the first `orelse` instruction will become known only after we translate the `body` block.

We could solve this problem if we move the instructions for each block into a separate data structure. Then, instead of specifying jump targets as concrete positions in the bytecode, we point to those data structures. Finally, when all blocks are translated and their sizes are know, we calculate arguments for jumps and assemble the blocks into a single sequence of instructions. And that's what the compiler does.

The blocks we're talking about are called basic blocks. They are not specific to CPython, though CPython's notion of a basic block differs from the conventional definition. According to the Dragon book, a basic block is a maximal sequence of instructions such that:

1. control may enter only the first instruction of the block; and
2. control will leave the block without halting or branching, except possibly at the last instruction.

CPython drops the second requirement. In other words, no instruction of a basic block except the first can be a target of a jump, but a basic block itself can contain jump instructions. To translate the AST from our example, the compiler creates four basic blocks:

1. instructions 0-14 for `test`
2. instructions 16-20 for `body`
3. instructions 22-24 for `orelse`; and
4. instructions 26-... for whatever comes after the if statement.

A basic block is represented by the `basicblock_` struct that is defined as follows:

```c
typedef struct basicblock_ {
    /* Each basicblock in a compilation unit is linked via b_list in the
       reverse order that the block are allocated.  b_list points to the next
       block, not to be confused with b_next, which is next by control flow. */
    struct basicblock_ *b_list;
    /* number of instructions used */
    int b_iused;
    /* length of instruction array (b_instr) */
    int b_ialloc;
    /* pointer to an array of instructions, initially NULL */
    struct instr *b_instr;
    /* If b_next is non-NULL, it is a pointer to the next
       block reached by normal control flow. */
    struct basicblock_ *b_next;
    /* b_seen is used to perform a DFS of basicblocks. */
    unsigned b_seen : 1;
    /* b_return is true if a RETURN_VALUE opcode is inserted. */
    unsigned b_return : 1;
    /* depth of stack upon entry of block, computed by stackdepth() */
    int b_startdepth;
    /* instruction offset for block, computed by assemble_jump_offsets() */
    int b_offset;
} basicblock;
```

And here's the definition of the `instr` struct:

```c
struct instr {
    unsigned i_jabs : 1;
    unsigned i_jrel : 1;
    unsigned char i_opcode;
    int i_oparg;
    struct basicblock_ *i_target; /* target block (if jump instruction) */
    int i_lineno;
};
```

We can see that the basic blocks are connected not only by jump instructions but also through the `b_list` and `b_next` fields. The compiler uses `b_list` to access all allocated blocks, for example, to free the memory. The `b_next` field is of more interest to us right now. As the comment says, it points to the next block reached by the normal control flow, which means that it can be used to assemble blocks in the right order. Returning to our example once more, the `test` block points to the `body` block, the `body` block points to the `orelse` block and the `orelse` block points to the block after the if statement. Because basic blocks point to each other, they form a graph called a [Control Flow Graph](https://en.wikipedia.org/wiki/Control-flow_graph) (CFG).

#### frame blocks

There is one more problem to solve: how to understand where to jump to when compiling statements like `continue` and `break`? The compiler solves this problem by introducing yet another type of block called frame block. There are different kinds of frame blocks. The `WHILE_LOOP` frame block, for example, points to two basic blocks: the `body` block and the block after the while statement. These basic blocks are used when compiling the `continue` and `break` statements respectively. Since frame blocks can nest, the compiler keeps track of them using stacks, one stack of frame blocks per code block. Frame blocks are also useful when dealing with statements such as `try-except-finally`, but we will not dwell on this now. Let's instead have a look at the definition of the `fblockinfo` struct:

```c
enum fblocktype { WHILE_LOOP, FOR_LOOP, EXCEPT, FINALLY_TRY, FINALLY_END,
                  WITH, ASYNC_WITH, HANDLER_CLEANUP, POP_VALUE };

struct fblockinfo {
    enum fblocktype fb_type;
    basicblock *fb_block;
    /* (optional) type-specific exit or cleanup block */
    basicblock *fb_exit;
    /* (optional) additional information required for unwinding */
    void *fb_datum;
};
```

We've identified three important problems and we've seen how the compiler solves them. Now, let's put everything together to see how the compiler works from the beginning to the end.

#### compiler units, compiler and assembler

As we've already figured out, after building a symbol table, the compiler performs two more steps to convert an AST to a code object:

1. it creates a CFG of basic blocks; and
2. it assembles a CFG into a code object.

This two-step process is performed for each code block in a program. The compiler starts by building the module's CFG and ends by assembling the module's CFG into the module's code object. In between, it walks the AST by recursively calling the `compiler_visit_*` and `compiler_*` functions, where `*` denotes what is visited or compiled. For example, `compiler_visit_stmt` delegates the compilation of a given statement to the appropriate `compiler_*` function, and the `compiler_if` function knows how to compile the `If` AST node. If a node introduces new basic blocks, the compiler creates them. If a node begins a code block, the compiler creates a new compilation unit and enters it. A compilation unit is a data structure that captures the compilation state of the code block. It acts as a mutable prototype of the code object and points to a new CFG. The compiler assembles this CFG when it exits a node that began the current code block. The assembled code object is stored in the parent compilation unit. As always, I encourage you to look at the struct definition:

```c
struct compiler_unit {
    PySTEntryObject *u_ste;

    PyObject *u_name;
    PyObject *u_qualname;  /* dot-separated qualified name (lazy) */
    int u_scope_type;

    /* The following fields are dicts that map objects to
       the index of them in co_XXX.      The index is used as
       the argument for opcodes that refer to those collections.
    */
    PyObject *u_consts;    /* all constants */
    PyObject *u_names;     /* all names */
    PyObject *u_varnames;  /* local variables */
    PyObject *u_cellvars;  /* cell variables */
    PyObject *u_freevars;  /* free variables */

    PyObject *u_private;        /* for private name mangling */

    Py_ssize_t u_argcount;        /* number of arguments for block */
    Py_ssize_t u_posonlyargcount;        /* number of positional only arguments for block */
    Py_ssize_t u_kwonlyargcount; /* number of keyword only arguments for block */
    /* Pointer to the most recently allocated block.  By following b_list
       members, you can reach all early allocated blocks. */
    basicblock *u_blocks;
    basicblock *u_curblock; /* pointer to current block */

    int u_nfblocks;
    struct fblockinfo u_fblock[CO_MAXBLOCKS];

    int u_firstlineno; /* the first lineno of the block */
    int u_lineno;          /* the lineno for the current stmt */
    int u_col_offset;      /* the offset of the current stmt */
};
```

Another data structure that is crucial for the compilation is the `compiler` struct, which represents the global state of the compilation. Here's its definition:

```C
struct compiler {
    PyObject *c_filename;
    struct symtable *c_st;
    PyFutureFeatures *c_future; /* pointer to module's __future__ */
    PyCompilerFlags *c_flags;

    int c_optimize;              /* optimization level */
    int c_interactive;           /* true if in interactive mode */
    int c_nestlevel;
    int c_do_not_emit_bytecode;  /* The compiler won't emit any bytecode
                                    if this value is different from zero.
                                    This can be used to temporarily visit
                                    nodes without emitting bytecode to
                                    check only errors. */

    PyObject *c_const_cache;     /* Python dict holding all constants,
                                    including names tuple */
    struct compiler_unit *u; /* compiler state for current block */
    PyObject *c_stack;           /* Python list holding compiler_unit ptrs */
    PyArena *c_arena;            /* pointer to memory allocation arena */
};
```

And the comment preceding the definition that explains what the two most important fields are for:

> The u pointer points to the current compilation unit, while units for enclosing blocks are stored in c_stack. The u and c_stack are managed by compiler_enter_scope() and compiler_exit_scope().

To assemble basic blocks into a code object, the compiler first has to fix the jump instructions by replacing pointers with positions in bytecode. On the one side, it's an easy task, since the sizes of all basic blocks are known. On the other side, the size of a basic block can change when we fix a jump. The current solution is to keep fixing jumps in a loop while the sizes change. Here's an honest comment from the source code on this solution:

> This is an awful hack that could hurt performance, but on the bright side it should work until we come up with a better solution.

The rest is straightforward. The compiler iterates over basic blocks and emits the instructions. The progress is kept in the `assembler` struct:

```c
struct assembler {
    PyObject *a_bytecode;  /* string containing bytecode */
    int a_offset;              /* offset into bytecode */
    int a_nblocks;             /* number of reachable blocks */
    basicblock **a_postorder; /* list of blocks in dfs postorder */
    PyObject *a_lnotab;    /* string containing lnotab */
    int a_lnotab_off;      /* offset into lnotab */
    int a_lineno;              /* last lineno of emitted instruction */
    int a_lineno_off;      /* bytecode offset of last lineno */
};
```

At this point, the current compilation unit and the assembler contain all the data needed to create a code object. Congratulations! We've done it! Almost.

#### peephole optimizer

The last step in the creation of the code object is to optimize the bytecode. This is a job of the peephole optimizer. Here's some types of optimizations it performs:

- The statements like `if True: ...` and `while True: ...` generate a sequence of `LOAD_CONST trueconst` and `POP_JUMP_IF_FALSE` instructions. The peephole optimizer eliminates such instructions.
- The statements like `a, = b,` lead to the bytecode that builds a tuple and then unpacks it. The peephole optimizer replaces it with a simple assignment.
- The peephole optimizer removes unreachable instructions after `RETURN`.

Essentially, the peephole optimizer removes redundant instructions, thus making bytecode more compact. After the bytecode is optimized, the compiler creates the code object, and the VM is ready to execute it.

### Summary

This was a long post, so it's probably a good idea to sum up what we've learned. The architecture of the CPython compiler follows a traditional design. Its two major parts are the frontend and the backend. The frontend is also referred to as the parser. Its job is to convert a source code to an AST. The parser gets tokens from the tokenizer, which is responsible for producing a stream of meaningful language units from the text. Historically, the parsing consisted of several steps, including the generation of a parse tree and the conversion of a parse tree to an AST. In CPython 3.9, the new parser was introduced. It's based on a parsing expression grammar and produces an AST straight away. The backend, also known paradoxically as the compiler, takes an AST and produces a code object. It does this by first building a symbol table and then by creating one more intermediate representation of a program called a control flow graph. The CFG is assembled into a single sequence of instructions, which is then optimized by the peephole optimizer. Eventually, the code object gets created.

At this point, we have enough knowledge to get acquainted with the CPython source code and understand some of the things it does. That's our plan for [the next time](https://tenthousandmeters.com/blog/python-behind-the-scenes-3-stepping-through-the-cpython-source-code/).