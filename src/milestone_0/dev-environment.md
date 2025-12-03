# 开发环境
我们将构建两个应用程序： 
1. 一个链上应用程序：一组部署在以太坊上的智能合约。 
1. 一个链下应用程序：一个与智能合约交互的前端应用程序。

虽然前端应用程序的开发是本书的一部分，但它并非本书的重点。我们构建前端应用程序仅仅是为了演示智能合约如何与前端应用程序集成。因此，前端应用程序是可选的，但我仍然会提供代码。

## 以太坊简介

以太坊是一个区块链，任何人都可以在上面运行应用程序。它看起来像一个云服务提供商，但两者之间存在诸多差异：
1. 您无需支付应用程序托管费用，但需要支付部署费用。 
2. 您的应用程序是不可变的，也就是说，部署后您将无法对其进行修改。 
3. 用户需要付费才能使用您的应用程序。

为了更好地理解这些时刻，让我们来看看以太坊是由什么构成的。

以太坊（以及任何其他区块链）的核心是数据库。以太坊数据库中最有价值的数据是账户状态。账户是一个以太坊地址及其关联数据：
1. 余额：账户的以太币余额。 
2. 代码：部署在此地址的智能合约的字节码。 
3. 存储空间：智能合约用于存储数据的空间。 
4. 随机数：用于防止重放攻击的序列整数。

以太坊的主要任务是以安全的方式构建和维护这些数据，防止未经授权的访问。

以太坊也是一个网络，一个由相互独立构建和维护状态的计算机组成的网络。该网络的主要目标是实现数据库访问的去中心化：不再有一个可英超修改任何数据的“权威机构”。这是通过共识机制实现的，共识机制是指网络中所有节点都必须遵守的一套规则。如果任何一方违反规则，将被排除在网络之外。

> 有趣的是：区块链可以使用 MySQL！除了性能之外，没有别的借口不用。而以太坊则使用 LevelDB，一个快速的键值数据库。

每个以太坊节点都运行着以太坊虚拟机（EVM）。虚拟机是一种可以运行其他程序的程序，而EVM则是一个执行智能合约的程序。用户通过交易与合约交互：除了简单的以太币发送之外，交易还可以包含智能合约调用数据。它包括：

1. 已编码的合约函数名称。 
2. 函数参数。

交易被打包成区块，然后由矿工挖出区块。网络中的每个参与者都可以验证任何交易和任何区块。

从某种意义上说，智能合约类似于 JSON API，但它不是通过接口调用，而是通过调用智能合约函数并提供函数参数来实现。与 API 后端类似，智能合约执行预先设定的逻辑，并且可以选择性地修改智能合约存储。与 JSON API 不同的是，要修改区块链状态，你需要发送一笔交易，并且每笔交易都需要支付一定的费用。

最后，以太坊节点公开了一个 JSON-RPC API。通过这个 API，我们可以与节点交互，例如：获取账户余额、估算 gas 费用、获取区块和交易记录、发送交易，以及在不发送交易的情况下执行合约调用（用于从智能合约中读取数据）。您可以在[这里](https://eth.wiki/json-rpc/API)找到所有可用端点的完整列表。

> 交易也可以通过 JSON-RPC API 发送，请参阅 [eth_sendTransaction](https://ethereum.org/en/developers/docs/apis/json-rpc/#eth_sendtransaction)

## 本地开发环境

目前有多种智能合约开发环境：
1. [Truffle](https://trufflesuite.com)
1. [Hardhat](https://hardhat.org)
1. [Foundry](https://github.com/foundry-rs/foundry)

Truffle是三者中最古老的，也是最不受欢迎的。Hardhat是它的改进版，也是应用最广泛的工具。Foundry是后起之秀，它带来了不同的测试视角。

虽然 HardHat 仍然是一种流行的解决方案，但越来越多的项目正在转向 Foundry。这其中有很多原因：
1. 使用 Foundry，我们可以用 Solidity 编写测试。这方便得多，因为我们在开发过程中无需在 JavaScript（Truffle 和 HardHat 使用 JS 进行测试和自动化）和 Solidity 之间来回切换。用 Solidity 编写测试也更加方便，因为它拥有所有原生特性（例如，无需为大数创建特殊类型，也无需在字符串和 BigNumber 之间进行转换）。
1. Foundry 在测试期间不会运行节点。这使得功能测试和迭代速度更快！Truffle 和 HardHat 会在每次运行测试时启动节点；而 Foundry 则在内部 EVM 上执行测试。

也就是说，我们将使用 Foundry 作为我们主要的智能合约开发和测试工具。

### Foundry

[Foundry](https://github.com/foundry-rs/foundry) 是一套用于以太坊应用程序开发的工具集。具体来说，我们将使用：
1. [Forge](https://github.com/foundry-rs/foundry/tree/master/forge), 一个 Solidity 的测试框架。
1. [Anvil](https://github.com/foundry-rs/foundry/tree/master/anvil), 这是一个专为 Forge 开发而设计的本地以太坊节点。我们将使用它把合约部署到本地节点，并通过前端应用程序连接到该节点。
1. [Cast](https://github.com/foundry-rs/foundry/tree/master/cast), 一款拥有大量实用功能的命令行工具。

Forge 让智能合约开发者的工作变得轻松许多。有了 Forge，我们无需运行本地节点来测试合约。Forge 会在其内部的 EVM 上运行测试，速度更快，而且无需发送交易和挖矿。

Forge 让我们可以用 Solidity 编写测试！Forge 也让模拟区块链状态变得更加容易：我们可以轻松地模拟以太币或代币余额，从其他地址执行合约，在任何地址部署任何合约等等。

但是，我们仍然需要一个本地节点来部署合约。为此，我们将使用 Anvil。前端应用程序使用 JavaScript Web3 库与以太坊节点交互（例如发送交易、查询状态、估算交易 gas 费用等）——这就是为什么我们需要运行本地节点的原因。

### Ethers.js

[Ethers.js](https://github.com/ethers-io/ethers.js/) 是一套用 JavaScript 编写的以太坊实用工具集。它是去中心化应用开发中最流行的两个 JavaScript 库之一（[web3.js](https://github.com/ChainSafe/web3.js))。这些库允许我们通过 JSON API 与以太坊节点进行交互，并且提供了许多实用函数，可以简化开发人员的工作。

### MetaMask

[MetaMask](https://metamask.io/) 是一款内置于浏览器中的以太坊钱包。它是一款浏览器扩展程序，可以创建并安全地存储私钥。MetaMask 是数百万用户使用的主要以太坊钱包应用程序。我们将使用它来签署要发送到本地节点的交易。

### React

[React](https://reactjs.org/) 是一个知名的 JavaScript 前端应用开发库。你不需要了解 React，我会提供一个应用模板。

## 项目设置

要设置项目，请创建一个新文件夹并在其中运行 `forge init` 命令：
```shell
$ mkdir uniswapv3clone
$ cd uniswapv3clone
$ forge init
```

> 如果您使用的是 Visual Studio Code，请在 `forge init` 命令中添加 `--vscode` 参数：`forge init --vscode`。Forge 将为 VSCode 进行特殊的适配。

Forge 会在 `src`, `test`和 `script` 文件夹中创建示例合约——这些合约可以删掉。

设置前端应用程序：
```shell
$ npx create-react-app ui
```

它位于子文件夹中，因此文件夹名称之间不存在冲突。
