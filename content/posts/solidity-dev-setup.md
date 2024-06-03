---
title: "Solidity Dev Setup"
date: 2024-06-03T19:22:21+08:00
draft: false
---

Solidity的开发工具和环境有几种：`Remix`, `HardHat`,`Foundry`。

Remix是很好的工具，初学者当然可以用了，Web IDE的优势就是打开就用。但是有几个问题，第一个是不够工程化不够高效，第二是多人协作不方便，其余比较主流比如VS Code + `HardHat` 提供了合约的开发，部署和测试等功能，最近又出现了`Foundry`基本类似于`Hardhat`, 但是最大的区别是测试以及`Foundry`的测试都使用Solidity编写，只要会Solidity就全搞定了。

对于我来说选择的自然是VS Code + `Hardhat` 的方式开发合约。先安装好VS Code（本身有的就不需要安装了），然后增加两个插件[`Solidity`](https://marketplace.visualstudio.com/items?itemName=NomicFoundation.hardhat-solidity)和`Solidity Extended` 前者要选Nomic Foundation提供的这个是`Hardhat`官方提供的插件(直接点击即可）。

然后安装node，npm和npx（一般装了node就有），然后用npm安装hardhat: `npm install --save-dev hardhat`, 使用`npx hardhat init` 进行项目的初始化期间会问一堆的问题，根据个人情况（是否会Typescript），因为选择了Typescript，项目中的各种脚本和脚手架都是用Typescript编写的，当然合约还是Solidity。init之后的合约全部在contracts目录下。

当合约编写完成后使用`npx hardhat compile`编译会生成artifacts, cache和ignition三个文件目录。合约的 abi 和 bytecode 等元数据会自动生成存放在 artifacts,  部署是通过 Ignition 模块定义的， 如果你创建了一个 TypeScript 项目，还会生成绑定目录typechain-types

基本上就是这样了。更多自行阅读`Hardhat`[文档](https://hardhat.org/hardhat-runner/docs/getting-started#quick-start)
