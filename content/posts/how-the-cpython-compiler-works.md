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