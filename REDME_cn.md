
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

![Front-end application screenshot](/screenshot.png)

We'll build a full clone of Uniswap V3. It **won't be an exact copy** and it **won't be production-ready** because we'll
do something in our own way and we'll **definitely** introduce multiple bugs. So, don't deploy this to the mainnet!

While our focus will primarily be on smart contracts, we'll also build a front-end application as a side hustle. 🙂
I'm not a front-end developer and I cannot make a front-end application better than you, but I can show you how a
decentralized exchange can be integrated into a front-end application.

The full code of what we'll build is stored in a separate repository:

https://github.com/Jeiwan/uniswapv3-code

You can read this book at:

https://uniswapv3book.com/

### Questions?

Each milestone has its own section in [the GitHub Discussions](https://github.com/Jeiwan/uniswapv3-book/discussions).
Don't hesitate to ask questions about anything that's not clear in the book!

## Table of Contents

- Milestone 0. Introduction
  1. Introduction to markets
  1. Constant Function Market Makers
  1. Uniswap V3
  1. Development Environment
  1. What We'll Build
- Milestone 1. First Swap
  1. Introduction
  1. Calculating Liquidity
  1. Providing Liquidity
  1. First Swap
  1. Manager Contract
  1. Deployment
  1. User Interface
- Milestone 2. Second Swap
  1. Introduction
  1. Output Amount Calculation
  1. Math in Solidity
  1. Tick Bitmap Index
  1. Generalize Minting
  1. Generalize Swapping
  1. Quoter Contract
  1. User Interface
- Milestone 3. Cross-tick Swaps
  1. Introduction
  1. Different Price Ranges
  1. Cross-Tick Swaps
  1. Slippage Protection
  1. Liquidity Calculation
  1. A Little Bit More on Fixed-point Numbers
  1. Flash Loans
  1. User Interface

- Milestone 4. Multi-pool Swaps
  1. Introduction
  1. Factory Contract
  1. Swap Path
  1. Multi-pool Swaps
  1. User Interface
  1. Tick Rounding
- Milestone 5. Fees and Price Oracle
  1. Introduction
  1. Swap Fees
  1. Flash Loan Fees
  1. Protocol Fees
  1. Price Oracle
  1. User Interface
- Milestone 6: NFT positions
  1. Introduction
  1. ERC721 Overview
  1. NFT Manager
  1. NFT Renderer

## Running locally

To run the book locally:
1. Install [Rust](https://www.rust-lang.org/).
1. Install [mdBook](https://github.com/rust-lang/mdBook):
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
1. Visit http://localhost:3000/ (or whatever URL the previous command outputs!)
