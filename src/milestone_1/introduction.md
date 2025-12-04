# 简介

在这个里程碑中，我们将构建一个资金池合约，它可以接收用户的流动性并在指定价格范围内进行交易。为了尽可能简化，我们将仅在一个价格范围内提供流动性，并且只允许单向交易。此外，我们将手动计算所有必要的数学运算，以便在开始使用 Solidity 中的数学库之前获得更好的直觉。

让我们来模拟一下我们将要构建的场景：
Let's model the situation we'll build:
1. 我们将创建一个 ETH/USDC 资金池合约。ETH 将作为 x 储备，USDC 将作为 y 储备。
1. 我们将当前价格设定为 1 ETH 兑换 5000 USDC。
1. 我们将提供流动性的价格区间为 1 ETH 兑换 4545-5500 USDC。
1. 我们将从资金池中购买一些 ETH。此时，由于我们只有一个价格区间，我们希望交易价格保持在价格区间内。

可视化一下，这个模型是这样的

![Buy ETH for USDC visualization](images/buy_eth_model.png)

在开始编写代码之前，我们先来梳理一下数学原理，计算模型的所有参数。为了简化操作，我将在用 Solidity 实现之前，先用 Python 进行数学计算。这样我们就可以专注于数学本身，而无需深入了解 Solidity 的具体数学运算。这也意味着，在智能合约中，我们将直接使用硬编码来表示所有数值。这样，我们就可以从一个简单易用的最小可行产品 (MVP) 开始着手。

为了方便起见，我把所有的 Python 计算都放在了 [unimath.py](https://github.com/Jeiwan/uniswapv3-code/blob/main/unimath.py)。

> You'll find the complete code of this milestone in [this Github branch](https://github.com/Jeiwan/uniswapv3-code/tree/milestone_1).

> If you have any questions, feel free to ask them in [the GitHub Discussion of this milestone](https://github.com/Jeiwan/uniswapv3-book/discussions/categories/milestone-1-first-swap)!