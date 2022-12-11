+++
title = "Contract Proxy patterns"
date = 2022-12-11T10:00:17+08:00
images = []
tags = ["EVM","Contract"]
categories = ["solidity"]
draft = false
+++

> [Proxy Patterns](https://blog.openzeppelin.com/proxy-patterns/)
> 注意原文成文时的solidity 版本为v0.4.21
> 思想可以参考实现可以略过


以太坊最大的优势之一是，每一笔移动资金的交易，每一份部署的合同，以及对合同进行的每一笔交易，在我们称为区块链的公共账本上都是不可改变的。
没有办法隐藏或修改任何曾经的交易。巨大的好处是，以太坊网络上的任何节点都可以验证每笔交易的有效性和状态，使以太坊成为非常强大鲁棒的去中心化系统。


但最大的缺点是，在你的智能合约被部署后，你不能改变它的源代码。从事中心化应用程序（如Facebook，或Airbnb）的开发人员习惯于频繁更新，
以修复错误或引入新功能。这在以太坊的传统模式下是不可能做到的。

还记得臭名昭著的Parity Wallet Multisig黑客攻击事件，15万ETH被盗？在这次攻击中，Parity multisig 钱包合约中的一个漏洞被利用了，
知名的钱包被抽走了资金。唯一可以做的应对是尝试比黑客更快，利用同样的漏洞入侵剩余的钱包，在攻击后将ETH重新分配给他们的合法主人。

如果有办法在智能合约部署后更新源代码就好了。。。

## Proxy patterns介绍

虽然不可能升级你已经部署的智能合约的代码，但可以设置一个代理合约架构，让你使用新部署的合约，就像你的主逻辑被升级了一样。

代理架构模式是这样的，所有的消息调用都要经过一个代理合约，它将把它们重定向到最新部署的合约逻辑。
为了升级，你的逻辑合约的一个新版本被部署，代理内保存的逻辑合约地址被更新以引用新的合约。
逻辑合约

[proxy-intro](/img/proxy-intro.svg)

Zeppelin一直在研究几种代理模式，作为他们实现zeppelin_os工作的一部分。探讨的三种方案是:

1. 继承存储(inherited Storage)
2. 永恒的存储(Eternal Storage)
3. 非结构化存储(Unstructured Storage)

这三种模式都依赖于低级别的委托调用。尽管Solidity提供了一个委托调用函数，但它只返回调用是否成功的true/false，不允许你管理返回的数据。

在我们深入研究之前，有两个重要的概念需要了解:

* 当一个合约对函数调用的函数不支持时，将调用fallback 函数。你可以写一个自定义的fallback 函数来处理这种情况。代理合约使用自定义fallback 函数来重定向对其他合约实现的调用。 
* 每当一个合约A 调用委托给另一个合约B时，它在合约A的上下文中执行合约B的代码，意味着msg.value和msg.sender的值将被保留，每一次存储修改都会影响合约A的存储。


[Zeppelin's Proxy contract](https://github.com/zeppelinos/labs/blob/master/upgradeability_using_eternal_storage/contracts/Proxy.sol) 的代理合约分享了所有代理模式，
它为这个特殊的原因实现了自己的委托调用函数，该函数返回调用逻辑合约的结果值。如果你打分享了使用Zeppelin的代理合约代码，你应该充分了解你要使用的代码。
让我们来探讨一下它到底是如何工作的，并了解, 它用来实现这一目的的汇编操作码。(请随时参考Solidity的[Assembly文档](https://docs.soliditylang.org/en/v0.4.21/assembly.html)以获得更多信息)

```
[assembly {
    let ptr := mload(0x40)
    calldatacopy(ptr, 0, calldatasize)
    let result := delegatecall(gas, _impl, ptr, calldatasize, 0, 0)
    let size := returndatasize
    returndatacopy(ptr, 0, size)
    switch result
    case 0 { revert(ptr, size) }
    default { return(ptr, size) }
 }
```

为了委托调用另一个solidity合约函数，我们必须把代理收到的msg.data传递给它。
由于msg.data是字节类型的，是一个动态的数据结构，它有一个变化的大小，存储在msg.data的第一个字（32 bytes）。
如果我们想只提取实际的数据，我们需要跨过第一个字的大小，从msg.data的`0x20` （32 bytes）开始。
然而，我们将利用两个操作码来代替这个操作。我们将使用`calldatasize` 来获取msg.data的大小，并使用`calldatacopy` 将其复制到我们的`ptr` 变量。

注意我们是如何初始化我们的ptr变量的。在Solidity中，位于`0x40` 位置的内存槽是特殊的，因为它包含了下一个可用的自由内存指针的值。
每次你直接保存一个变量到内存，你应该通过检查 `0x40` 的值来咨询你应该把它保存到哪里。现在我们知道了允许我们保存变量的位置，
我们可以使用`calldatacopy` 将大小为`calldatasize` 的`calldata` 从调用数据的`0` 开始复制到`ptr` 的位置。

```
let ptr := mload(0x40)
calldatacopy(ptr, 0, calldatasize)
```

让我们看看汇编块中使用`delegatecall` 操作码的下面一行。

```
let result := delegatecall(gas, _impl, ptr, calldatasize, 0, 0)
```
** Parameters ** 

* gas 我们传入执行函数所需的gas
* `_impl` 我们要调用的逻辑合约的地址
* `ptr` 数据开始位置的内存指针
* `calldatasize` 我们要传递的数据的大小
* 0，代表调用逻辑合约后的返回值的数据。这是未使用的，因为我们还不知道数据的大小，因此不能把它分配给一个变量。我们仍然可以在以后使用 `returndata` 操作码访问这一信息
* 0, 为返回值大小。这是未使用的，因为我们没有机会创建一个临时变量来存储数据出来，因为我们在调用其他合约之前不知道它的大小。我们可以用另一种方式获得这个值，即在后面调用`returndatasize` 操作码

下一行使用 `returndatasize` 操作码抓取返回数据的大小

```
let size := returndatasize
```

我们使用返回数据的大小来复制返回数据的内容到我们的 `ptr` 变量上，并使用一个辅助操作码函数 `returndatacopy`

```
returndatacopy(ptr, 0, size)
```

最后，switch语句要么返回返回的数据，要么在出错时抛出一个异常。

[proxy-data-flow](/img/proxy-data-flow.svg)

很好，我们现在有办法从逻辑合同中检索到适当的结果值。

现在我们了解了代理合约的工作原理，让我们看看Zeppelin提出的三种模式。使用继承存储的可升级性、非结构化存储和永恒的存储。

这三种方法有不同的方式来解决相同的技术难题：如何确保逻辑合约不会覆盖用于可升级性代理的状态变量。

任何代理架构模式的主要问题是如何处理存储分配。请记住，由于我们是用一个合
约来处理存储，另一个合约来处理逻辑，任何一个合约都有可能覆盖一个已经使用过的存储槽。
这意味着，如果代理合约有一个状态变量来跟踪某个存储槽的最新逻辑合约地址，而逻辑合约不知道，那么逻辑合约就可能在同一槽中存储一些其他数据，从而覆盖代理的关键信息。
Zeppelin的三种方法提出了不同的方法来架构你的系统，使你的合约可以通过代理模式来升级。

## 使用继承存储（Inherited Storage）升级
[继承存储](https://github.com/OpenZeppelin/openzeppelin-labs/tree/master/upgradeability_using_inherited_storage)的方法依赖于使逻辑合约包含代理所需的存储结构。代理和逻辑合约都继承了相同的存储结构，以确保两者都坚持存储必要的代理状态变量。

在探索这种方法的时候，我们尝试了由一个注册表合约来跟踪你的逻辑合约的不同版本的想法。为了升级到一个新的逻辑合约，你需要在注册表中把它注册为一个新的版本，并要求代理升级到它。请注意，拥有一个注册表并不影响存储机制；事实上，它可以用本帖中显示的任何一种存储模式来实现。


[inherited Storage](/img/inherited-storage.png)

### 如何初始化

1. 部署Registry合约
2. 部署你的合约的初始版本（v1）。确保它继承了可升级合约
3. 将你的初始版本的地址注册到`Registry`
4. 要求 `Registry` 合约创建一个 `UpgradeabilityProxy` 实例
5. 调用你的 `UpgradeabilityProxy` 以升级到合约的初始版本

### 如何升级

1. 部署一个新版本的合约（v2），它继承了你的初始版本，以确保它保持代理的存储结构和初始版本的合约中的存储结构。
2. 将你的新版合同注册到 `Registry`
3. 调用你的 `UpgradeabilityProxy` 以升级到合约的新注册版本
 

### 总结
我们可以在未来部署的逻辑合约中引入升级的函数以及新的函数和新的状态变量，方法是仍然调用相同的 `UpgradeabilityProxy` 合约


## 使用永恒存储（Eternal Storage）升级
在[永恒存储](https://github.com/OpenZeppelin/openzeppelin-labs/tree/master/upgradeability_using_eternal_storage) 模式中，存储模式被定义在一个单独的合约中，代理和逻辑合约都继承于此。存储合约持有逻辑合约需要的所有状态变量，
由于代理也知道这些变量，所以它可以定义自己的可升级性所需的状态变量，而不用担心它们被覆盖。
请注意，所有未来版本的逻辑合同不应该定义任何其他的状态变量。所有版本的逻辑合约必须始终使用一开始定义的永恒的存储结构。

Zeppelin lab的代码库中提供的这个实现也引入了代理所有权的概念。代理所有者是唯一能够升级代理以指向新的逻辑合约的地址，也是唯一能够转移所有权的地址。

[eternal proxy](/img/eternal-proxy.png)

### 如何初始化

1. 部署 `EternalStorageProxy`
2. 部署合约的初始版本 (v1)
3. 调用你的 `EternalStorageProxy` 实例来升级到你的初始版本的地址
4. 如果你的逻辑合约依赖它的构造函数来设置一些初始状态，那就必须在它链接到代理后重做，因为代理的存储不知道这些值。`EternalStorageProxy有` 一个函数 `upgradeToAndCall`，专门用来调用你的逻辑合约上的一些函数，在代理升级到它之后重新进行设置。

### 如何升级

1. 部署一个新版本的合约(v2)， 确保它拥有永恒的存储结构。
2. 调用你的`EternalStorageProxy` 实例来升级到新版本。

### 总结

直观的方法，对代币逻辑合约没有明显的开销。未来的逻辑接触可以升级现有的方法和引入新的方法，但不应该引入新的状态变量。

## 使用非结构化存储（Unstructured Storage）升级

[非结构化存储模式](https://github.com/OpenZeppelin/openzeppelin-labs/tree/master/upgradeability_using_unstructured_storage) 与继承性存储相似，但不要求逻辑契约继承任何与可升级性相关的状态变量。这种模式使用代理合同中定义的非结构化存储槽来保存可升级性所需的数据。

在代理合约中，我们定义了一个常量变量，当哈希运算时，应该给出一个足够随机的存储位置来存储代理应该调用的逻辑合同的地址。

```
bytes32 private constant implementationPosition = 
keccak256("org.zeppelinos.proxy.implementation");
```

由于[常量状态变量](http://solidity.readthedocs.io/en/v0.4.21/miscellaneous.html#modifiers) 不占用存储槽，所以不存在实现位置被逻辑合约意外覆盖的问题。
由于Solidity它的[状态变量](http://solidity.readthedocs.io/en/v0.4.21/miscellaneous.html#layout-of-state-variables-in-storage)存储中的布局方式，这个存储槽被逻辑合约中定义的其他东西使用的碰撞机会极少。

通过使用这种模式，没有一个逻辑合约版本必须知道代理的存储结构，然而所有未来的逻辑合约必须继承其祖先版本所声明的存储变量。就像在继承存储模式中，未来升级的代币逻辑合约可以升级现有的功能，也可以引入新的功能和新的存储变量。

Zeppelin lab的代码库中提供的这个实现也引入了代理所有权的概念。代理所有者是唯一能够升级代理以指向新的逻辑合约的地址，也是唯一能够转移所有权的地址。

[unstructured proxy](/img/unstructured-proxy.png)

### 如何初始化

1. 部署 `OwnedUpgradeabilityProxy`实例
2. 部署合约的初始版本 (v1)
3. 调用你的 `OwnedUpgradeabilityProxy` 实例来升级到你的初始版本的地址
4. 如果你的逻辑合约依赖它的构造函数来设置一些初始状态，那就必须在它链接到代理后重做，因为代理的存储不知道这些值。`OwnedUpgradeabilityProxy` 有一个函数 `upgradeToAndCall`，专门用来调用你的逻辑合约上的一些函数，在代理升级到它之后重新进行设置。

### 如何升级

1. 部署一个新版本的合约(v2)， 确保它继承了以前版本中使用的状态变量结构。
2. 调用你的`OwnedUpgradeabilityProxy` 实例来升级到新版本。

### 总结

这种方法很好，因为它不需要Token逻辑合约意识到它是代理合约系统的一部分。
