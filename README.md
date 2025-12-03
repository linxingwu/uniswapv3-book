
# Uniswap V3 Development Book

<p align="center">
<img src="/src/images/cover.png" alt="Uniswap V3 Development Book cover" width="360"/>
</p>


<p align="center">
👉&nbsp;<a href="https://uniswapv3book.com/">在线阅读</a>&nbsp;&nbsp;|&nbsp;&nbsp;<a href="https://uniswapv3book.com/print.html">另存为PDF</a>&nbsp;👈
</p>

这本书将教会你如何开发一个高级的去中心化的应用！具体来说，我们将会创建一个 
[Uniswap V3](https://uniswap.org/)的拷贝, 这是一个去中心化的交易所。

## 为什么选择 Uniswap?
- 实现了一个很简单的数学原理，`x*y=k`，这个公示现在还有无比的潜力。
- 这是一个在简单公式基础上叠了一层厚厚工程的高级应用程序。
- 它无需许可，而且久经考验。学习一个在生产环境中运行数年处理了百万交易的程序会让你成为一个更好的开发者。

## 我们会做什么

![前端向明截图](/screenshot.png)

我们会创建一个 Uniswap V3 的副本. 我们 **不会直接拷贝原项目** 也 **不足以部署到生产环境** 因为我们会以自己的方式实现一些地方。我们**肯定**会引入很多bug，不要直接把本项目部署在主网上！

虽然我们的主要重点是智能合约，但我们也会兼顾前端应用的开发。🙂我不是前端开发人员，我不可能做出比你更好的前端应用，但我可以向你展示如何将去中心化交易所集成到前端应用中。

我们将要构建的全部代码都存储在一个单独的代码库中：

https://github.com/Jeiwan/uniswapv3-code

您可以在以下网址阅读这本书：

https://uniswapv3book.com/

### 有疑问?
每个里程碑在 [the GitHub Discussions](https://github.com/Jeiwan/uniswapv3-book/discussions)都有自己的版块。如果书中有任何不清楚的地方，请随时提问！

## 目录

- 里程碑 0. 简介
  1. 市场概论
  1. 恒定函数做市商
  1. Uniswap V3
  1. 开发环境
  1. 我们将会做什么
- 里程碑 1. 第一个交换
  1. 简介
  1. 流动性计算
  1. 提供流动性
  1. 第一个交换
  1. 管理合约
  1. 部署
  1. 用户接口
- 里程碑 2. 第二个交换
  1. 简介
  1. 输出数量计算
  1. Solidity中的数学
  1. Tick Bitmap 索引
  1. 一般铸币
  1. 一般交换
  1. 引用合约
  1. 用户接口
- 里程碑 3. 跨tick 交换
  1. 简介
  1. 不同价格区间
  1. 跨tick 交换
  1. 防滑保护
  1. 流动性计算
  1. 定点数番外
  1. 闪电贷
  1. 用户接口

- 里程碑 4. 跨池交换
  1. 简介
  1. 工厂合约
  1. 交换路径
  1. 跨池交换
  1. 用户接口
  1. Tick 舍入
- 里程碑 5. 交易费和价格预测
  1. 简介
  1. 交易费
  1. 闪电贷费
  1. 协议费
  1. 价格预测
  1. 用户接口
- 里程碑 6: NFT 头寸
  1. 简介
  1. ERC721 简介
  1. NFT 管理器
  1. NFT 渲染器

## 本地运行

本地运行:
1. 安装 [Rust](https://www.rust-lang.org/).
1. 安装 [mdBook](https://github.com/rust-lang/mdBook):
    ```shell
    $ cargo install mdbook
    $ cargo install mdbook-katex
    ```
1. Clone the repo:
    ```shell
    $ git clone https://github.com/Jeiwan/uniswapv3-book
    $ cd uniswapv3-book
    ```
1. Run:
    ```shell
    $ mdbook serve --open
    ```
1. 打开 http://localhost:3000/ (或者命令行输出的其他URL!)
