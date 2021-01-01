+++
title = "Python的幕后#1: CPython VM 是如何工作的"
date = 2021-01-01T10:20:20+08:00
images = []
tags = ["translate"]
categories = ["python"]
draft = false

+++

> [原文](https://tenthousandmeters.com/blog/python-behind-the-scenes-1-how-the-cpython-vm-works/) 
>
> 本文已获原文作者[Victor Skvortsov](https://tenthousandmeters.com/about/)授权

## 引言

你是否想过 当你用`python`命令执行你的程序的时候, 它做了什么?
```shell
$ python script.py
```
这篇文章是一系列试图回答这个问题文章的开端。我们将会深入CPython的内部(这是Python最流行的实现)。这样我们将更深入的了解语言本身。这是本系列文章的主要目的。如果你熟悉Python和C，但是并不熟悉CPython源码，那么你可能你会发现本文很有趣。

## 什么是CPython并且为什么有人想要学习它呢?

让我们从众所周知的地方开始谈起。 CPython是C语言实现的Python解释器。它是Python的一种实现，其他的实现有PyPy，Jython，IronPython等等。CPython是最原始，维护最久和使用最广的一种实现。

CPython实现了Python，但是什么是Python? 一个简单的回答是：Python是一种编程语言。当更近一步提出相同含义的问题时，答案变得更加具体：什么定义了Python是什么？ Python与C之类的语言不同，它没有正式的规范。 最接近它的是Python语言参考，它开始是以下内容:

>当我尝试尽可能精确时，我选择对语法和词法分析以外的所有内容使用英语而不是正式的规范。 这应该使普通读者更容易理解文档，但会存在歧义。 因此，如果您是来自火星并试图仅通过本文档重新实现Python，则您可能不得不猜测，实际上您可能最终会实现完全不同的语言。 另一方面，如果您正在使用Python，并且想知道关于该语言特定区域的确切规则是什么，那么您肯定可以在这里找到它们。

因此，Python并非仅由其语言参考来定义。说Python是由其参考实现CPython定义的，这也是错误的，因为有些实现细节不是该语言的一部分。依赖引用计数的垃圾收集器就是一个例子。 由于没有单一的事实来源，因此我们可以说Python的一部分是由Python语言参考定义的，另一部分是由其主要实现CPython定义的。

这种说法似乎有些古怪，但我认为弄清这个问题对我们将要研究的主题至关重要。 但是，您可能仍然想知道为什么我们应该研究它。 除了好奇心外，我还发现以下原因:

* 纵览全貌可以更深入地了解该语言。 如果您了解Python的某些实现细节，那么掌握Python的某些特性就容易得多。

* 语言实现细节在实践中很重要。 当人们想了解语言的适用性及其局限性，估计性能或检测对效率的影响时，对象的存储方式，垃圾收集器的工作方式以及如何协调多个线程是非常重要的主题。

* CPython提供了Python / C API，该API允许使用C扩展Python并将Python嵌入C中。要有效地使用此API，程序员需要对CPython的工作方式有充分的了解。

## 了解CPython如何工作需要什么?

CPython的设计易于维护。 新手当然可以期望能够阅读源代码并了解其功能。 但是，可能需要一些时间。 通过本系列文章，希望对您有所帮助。

## 该系列的讲解方式

我选择采取自上而下的方式。 在这一部分中，我们将探讨CPython虚拟机（VM）的核心概念。 接下来，我们将了解CPython如何将python源代码编译为VM可以执行的程序。 之后，我们将熟悉CPython源代码，并逐步执行一个程序，在此过程中研究解释器的主要部分。 最终，我们将能够逐一熟悉出语言的不同方面，并查看它们是如何实现的。 这绝不是一个严格的计划，而是我的大概想法。

Note: 在这篇文章中，我指的是CPython 3.9。 随着CPython的发展，某些实现细节肯定会发生变化。 我将尝试跟踪重要的更改并添加更新说明。

## 概览

Python程序的执行大致包括三个阶段:

1. 初始化
2. 编译
3. 解释

在初始化阶段，CPython将初始化运行Python所需的数据结构。 它还准备诸如内置类型，配置和加载内置模块，设置导入系统等功能。 这是一个非常重要的阶段，由于其服务的性质，CPython的探索者经常忽略它。

接下来是编译阶段。 从不产生机器代码的意义上讲，CPython是解释器，而不是编译器。 但是，解释器通常在执行之前将源代码转换为某种中间表示。 CPython也是如此。 此翻译阶段执行的操作与典型编译器相同：解析源代码并构建AST（抽象语法树），从AST生成字节码，甚至执行一些字节码优化。

在进行下一阶段之前，我们需要了解什么是字节码。 字节码是一系列指令。 每条指令由两个字节组成：一个字节用于操作码，一个字节用于参数。 考虑一个例子：

```python
def g(x):
  return x + 3
```

CPython将函数`g()`的函数体转换为以下字节序列：[124，0，100，1，23，0，83，0]。 如果我们运行标准库`dis`对其进行反汇编，则将获得以下信息：

```shell
$ python -m dis example1.py
...
2           0 LOAD_FAST            0 (x)
            2 LOAD_CONST           1 (3)
            4 BINARY_ADD
            6 RETURN_VALUE
```

LOAD_FAST操作码对应于字节124，参数为0。LOAD_CONST操作码对应于字节100，参数为1。BINARY_ADD和RETURN_VALUE指令始终分别编码为（23，0）和（83，0），因为他们不需要参数。

CPython的核心是执行字节码的虚拟机。 通过查看前面的示例，您可能会猜测它是如何工作的。 CPython的VM是基于堆栈的。 这意味着它使用堆栈执行指令来存储和检索数据。 LOAD_FAST指令将局部变量压入堆栈。 LOAD_CONST将一个常数压栈。 BINARY_ADD从堆栈中弹出两个对象，将它们加起来并将结果压回去。 最后，RETURN_VALUE弹出堆栈中的所有内容，并将结果返回给其调用方。

字节码在一个巨大的循环(evaluation loop)中执行，该循环在有指令时运行。 它将在产生值或产生错误时停止。

这样的简短概述会引发很多疑问：

* LOAD_FAST和LOAD_CONST操作码的参数是什么意思？ 他们是指数吗？ 他们索引什么？
* VM是否在堆栈上放置值或对对象的引用？
* CPython如何知道x是局部变量？
* 如果参数太大而无法容纳单个字节怎么办？
* 将两个数字相加的指令是否与连接两个字符串相同？ 如果是，那么VM如何区分这些操作？

为了回答这些以及其他有趣的问题，我们需要研究CPython VM的核心概念。

## 代码对象，函数对象，帧对象

### 代码对象

我们看到了一个简单函数的字节码的样子。 但是典型的Python程序更加复杂。 VM如何执行包含功能定义并进行功能调用的模块？

看下面的代码思考下:

```python
def f(x):
    return x + 1

print(f(1))
```

它的字节码是什么样的？ 为了回答这个问题，让我们分析一下程序的功能。 它定义了函数`f`()，以1作为参数调用`f()`并输出调用结果。 无论函数`f()`做什么，它都不是模块字节码的一部分。 我们可以通过运行反汇编程序来确认这一点。

```python
$python -m dis example2.py
1           0 LOAD_CONST               0 (<code object f at 0x10bffd1e0, file "example.py", line 1>)
            2 LOAD_CONST               1 ('f')
            4 MAKE_FUNCTION            0
            6 STORE_NAME               0 (f)

4           8 LOAD_NAME                1 (print)
           10 LOAD_NAME                0 (f)
           12 LOAD_CONST               2 (1)
           14 CALL_FUNCTION            1
           16 CALL_FUNCTION            1
           18 POP_TOP
           20 LOAD_CONST               3 (None)
           22 RETURN_VALUE
...
```

在第1行上，我们通过从称为代码对象的对象制作函数并将其绑定到名称来定义函数`f()`。 我们看不到函数`f()`的字节码返回递增的参数。模块或函数体之类的单个可执行的代码段称为代码块， CPython将有关代码块功能的信息存储在称为代码对象的结构中， 它包含字节码以及该块内使用的变量名称列表之类的内容。 运行模块或调用函数意味着开始执行相应的代码对象。

### 函数对象

但是，函数不仅是代码对象。 它必须包括其他信息，例如函数名称，文档字符串，默认参数以及在作用域内定义的变量的值。 此信息与代码对象一起存储在功能对象中。 MAKE_FUNCTION指令用于创建它。 在CPython源代码中对函数对象结构的定义前面带有以下注释：

>函数对象和代码对象不应相互混淆:
>
>函数对象是通过执行`def`语句创建的。他们在其`__code__`属性中引用了一个代码对象，该对象是纯粹的语法对象，即仅是某些源代码行的编译版本。每个源代码“片段”有一个代码对象，但是每个代码对象可以被零个或多个函数对象引用，这取决于到目前为止，源代码中的“ def”语句执行了多少次。
>

几个函数对象如何引用一个代码对象？ 这是一个例子：

```python
def make_add_x(x):
    def add_x(y):
        return x + y
    return add_x

add_4 = make_add_x(4)
add_5 = make_add_x(5)
```

`make_add_x()`函数的字节码包含MAKE_FUNCTION指令。函数`add_4()`和`add_5()`是使用相同的代码对象调用此指令的结果。但是有一个不同的参数– x的值。每个函数都通过[cell variables](https://tenthousandmeters.com/blog/python-behind-the-scenes-5-how-variables-are-implemented-in-cpython/)获得自己的功能，该机制使我们能够创建诸如`add_4()`和`add_5()`之类的闭包。

在讲述下一个概念之前, 让我们先来查看代码和函数对象的C定义，以更好地了解它们的含义。

```c
struct PyCodeObject {
    PyObject_HEAD
    int co_argcount;            /* #arguments, except *args */
    int co_posonlyargcount;     /* #positional only arguments */
    int co_kwonlyargcount;      /* #keyword only arguments */
    int co_nlocals;             /* #local variables */
    int co_stacksize;           /* #entries needed for evaluation stack */
    int co_flags;               /* CO_..., see below */
    int co_firstlineno;         /* first source line number */
    PyObject *co_code;          /* instruction opcodes */
    PyObject *co_consts;        /* list (constants used) */
    PyObject *co_names;         /* list of strings (names used) */
    PyObject *co_varnames;      /* tuple of strings (local variable names) */
    PyObject *co_freevars;      /* tuple of strings (free variable names) */
    PyObject *co_cellvars;      /* tuple of strings (cell variable names) */

    Py_ssize_t *co_cell2arg;    /* Maps cell vars which are arguments. */
    PyObject *co_filename;      /* unicode (where it was loaded from) */
    PyObject *co_name;          /* unicode (name, for reference) */
        /* ... more members ... */
};
```

```c
typedef struct {
    PyObject_HEAD
    PyObject *func_code;        /* A code object, the __code__ attribute */
    PyObject *func_globals;     /* A dictionary (other mappings won't do) */
    PyObject *func_defaults;    /* NULL or a tuple */
    PyObject *func_kwdefaults;  /* NULL or a dict */
    PyObject *func_closure;     /* NULL or a tuple of cell objects */
    PyObject *func_doc;         /* The __doc__ attribute, can be anything */
    PyObject *func_name;        /* The __name__ attribute, a string object */
    PyObject *func_dict;        /* The __dict__ attribute, a dict or NULL */
    PyObject *func_weakreflist; /* List of weak references */
    PyObject *func_module;      /* The __module__ attribute, can be anything */
    PyObject *func_annotations; /* Annotations, a dict or NULL */
    PyObject *func_qualname;    /* The qualified name */
    vectorcallfunc vectorcall;
} PyFunctionObject;

```

### 帧对象

VM执行代码对象时，必须跟踪变量的值和不断变化值的堆栈。它还需要记住在哪里停止执行当前代码对象以执行另一个代码对象，以及在哪里返回。 CPython将此信息存储在帧对象或简单的帧中。 帧提供了代码对象可以执行需要的状态。 为了让我们对CPython源代码会越来越熟悉，我在这里也保留了帧的定义：

```c
struct _frame {
    PyObject_VAR_HEAD
    struct _frame *f_back;      /* previous frame, or NULL */
    PyCodeObject *f_code;       /* code segment */
    PyObject *f_builtins;       /* builtin symbol table (PyDictObject) */
    PyObject *f_globals;        /* global symbol table (PyDictObject) */
    PyObject *f_locals;         /* local symbol table (any mapping) */
    PyObject **f_valuestack;    /* points after the last local */

    PyObject **f_stacktop;          /* Next free slot in f_valuestack.  ... */
    PyObject *f_trace;          /* Trace function */
    char f_trace_lines;         /* Emit per-line trace events? */
    char f_trace_opcodes;       /* Emit per-opcode trace events? */

    /* Borrowed reference to a generator, or NULL */
    PyObject *f_gen;

    int f_lasti;                /* Last instruction if called */
    /* ... */
    int f_lineno;               /* Current line number */
    int f_iblock;               /* index in f_blockstack */
    char f_executing;           /* whether the frame is still executing */
    PyTryBlock f_blockstack[CO_MAXBLOCKS]; /* for try and loop blocks */
    PyObject *f_localsplus[1];  /* locals+stack, dynamically sized */
};

```

第一个帧在执行模块的代码对象时创建。 每当需要执行另一个代码对象，CPython都会创建一个新帧。 每个帧都有对前一帧的引用。 因此，帧形成帧的栈，也称为调用栈，当前帧位于顶部。 调用函数时，会将新的帧压入栈。 从当前执行的帧返回时，CPython通过记住其最后处理的指令来继续执行前一帧。 从某种意义上来说，CPython VM除了构造和执行帧外什么也不做。 但是，正如我们将很快看到的那样，这个描述，简单来说说，隐藏了一些细节。

### 线程, 解释器, 运行时

我们已经研究了三个重要概念：

* 代码对象
* 函数对象
* 帧对象

CPython还有三个别的:

* 线程状态
* 解释器状态
* 运行时状态

#### 线程状态

线程状态是一种数据结构，其中包含了线程特有的数据，包括调用堆栈，异常状态和调试设置。

不应将其与操作系统的线程混淆。但是它们有非常紧密的关系，考虑使用标准库 [`treading`](https://docs.python.org/3/library/threading.html) 模块在单独的线程里运行函数会发生什么:

```python
from threading import Thread

def f():
    """Perform an I/O-bound task"""
    pass

t = Thread(target=f)
t.start()
t.join()
```

`t.start()` 实际上通过调用操作系统的函数(Unix/Linux系统上是`pthread_create()`，Windnows上是`_beginthreadex()` )创建了一个新的系统线程. 新创建的线程从`_thread`模块中调用负责调用目标的函数.

新创建的线程通过`_thread`模块中的函数调用该目标函数。该函数不仅接收目标函数和目标函数的参数，还接收要在新操作系统线程中使用的新线程状态。新操作系统进程将执行循环放入自己的线程状态里， 因此始终可以使用它。

我们可能还记得著名的GIL（全局解释器锁) 它防止多个线程同时进入执行循环。这样做的主要原因是在不引入更多细粒度的锁的情况下保护CPython的状态免受损坏。 [Python/C API 指南](https://docs.python.org/3.9/c-api/index.html) 清楚的解释了GIL:

>*Python解释器不是完全线程安全的。Python解释器不是完全线程安全的。 为了支持多线程Python程序，有一个全局锁，称为全局解释器锁或GIL，必须由当前线程持有，然后才能安全地访问Python对象。 如果没有锁，即使是最简单的操作也可能在多线程程序中引起问题：例如，当两个线程同时增加同一对象的引用计数时，引用计数最终只能被增加一次，而不是两次。*

要管理多个线程，需要有一个比线程状态更高级别的数据结构。

#### 解释器和运行时状态

实际上，上面说的更高级的数据结构有两个：解释器状态和运行时状态。 两者的差别似乎并不十分明显。

解释器状态是一组线程以及该组特定的数据. 线程共享诸如加载的模块(`sys.modules`), , 内置模块(`builtins.__dict__`)以及导入系统之类的东西 (`importlib`)。

运行时状态是全局变量。它存储特定进程的数据。 其中包括CPython的状态（例如，是否已初始化？）和GIL机制。

通常，一个进程的所有线程都属于同一个解释器。 T但是，在少数情况下，可能需要创建一个子解释器来隔离一组线程。一个例子是 [mod_wsgi](https://modwsgi.readthedocs.io/en/develop/user-guides/processes-and-threading.html#python-sub-interpreters) ，它使用不同的解释器来运行WSGI应用程序。隔离的最明显效果是每组线程都有自己的所有模块版本，包括`__main__`，这是一个全局命名空间。

CPython没有提供类似于`threading`模块的简便方法来创建新的解释器。目前仅通过Python/C API提供支持,  [但有一天可能会更改](https://www.python.org/dev/peps/pep-0554/) 。

### 架构摘要

让我们快速总结一下CPython的体系结构，看看一切如何融合在一起。解释器可以看作是分层结构。 以下总结了这些层是什么:

1. 运行时: 进程的整体状态；这包括GIL和内存分配机制
2. 解释器: 一组线程和它们共享的一些数据，例如导入的模块。
3. 线程：包含特定数据的操作系统线程； 包括调用堆栈。
4. 帧：调用堆栈的元素； 帧包含一个代码对象，并提供执行它的状态。
5. 执行循环：执行帧对象的地方。

这些层由我们已经看到的相应数据结构表示。 在某些情况下，它们并不等价。 例如，使用全局变量来实现内存分配机制。 它不是运行时状态的一部分，但肯定是运行时层的一部分。

### 结论

在这一部分中，我们概述了`python` 命令为执行Python程序所做的工作。 我们已经看到它在三个阶段起作用:

1. 初始化CPython
2. 将源代码编译为模块的代码对象； 和
3. 执行代码对象的字节码。

解释器中负责字节码执行的部分称为虚拟机。 CPython VM具有几个特别重要的概念：代码对象，帧对象，线程状态，解释器状态和运行时。 这些数据结构构成了CPython体系结构的核心。

我们没有涉及很多东西。 我们避免深入研究源代码。 初始化和编译阶段完全超出了我们的范围。 相反，我们从虚拟机的概述开始。 我认为，通过这种方式，我们可以更好地了解每个阶段的职责。 现在，我们知道了CPython将源代码编译到的代码对象。

[下一篇](https://tenthousandmeters.com/blog/python-behind-the-scenes-2-how-the-cpython-compiler-works/)，我们将看到它是如何做到的。

