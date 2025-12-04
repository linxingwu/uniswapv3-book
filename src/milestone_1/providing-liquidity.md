# 提供流动性

理论到此为止，开始写代码!

创建一个新的文件夹 (我建的叫 `uniswapv3-code`), 然后运行 `forge init --vscode` –初始化一个 Forge 项目。  `--vscode` 标志告诉Forge为项目配置Solidity拓展。

把默认合约和测试文件删掉：
- `script/Contract.s.sol`
- `src/Contract.sol`
- `test/Contract.t.sol`

现在开始，我们来建立我们的第一个智能合约!

## 合约池

你之前学过， Uniswap 部署了多个合约池，每一个池子都是一对代币的交易所。Uniswap 把所有的合约分为两类：

- 核心合约
- 外围合约

核心合约，代表了核心逻辑的实现。简练，用户不友好，底层。他们只有一个任务，就是尽可能保证安全可靠。在 Uniswap 3里，有两种这样的合约：
1. 合约池，实现了去中心化交易所的核心逻辑。
2. 工厂合约，作为合约池的管理器，使得部署合约池更为方便。

我们从合约池开始，实现99%的Uniswap功能。
We'll begin with the pool contract, which implements 99% of the core functionality of Uniswap.

建立 `src/UniswapV3Pool.sol`:

```solidity
pragma solidity ^0.8.14;

contract UniswapV3Pool {}
```
我们思考一下合约会存储的数据：
1. 既然每一个合约池都是一个代币对的交易所，我们需要保持两个代币地址。这些地址是静态的，在合约池部署时设置一次不会再改变（也就是不可变的。）
1. 每个合约池都是一组流动性头寸。我们将它们存储在一个map中，其中key是唯一的头寸标识符，value是包含头寸信息的结构体。
1. 每个合约池还需要维护一个 tick 注册表——也是一个map，其中key是 tick 索引，值是存储 tick 信息的结构体。
1. 因为tick是有限的，我们需要用常量来存下这个范围值。
1. 合约池存储的流动性$L$,我们需要一个变量来存储。
1. 最后，我们需要跟踪当前价格及其对应的 tick 值。为了优化 gas 消耗，我们会将它们存储在同一个存储槽中：这些变量通常会被同时读写，很适合使用 Solidity 的状态变量打包。

总而言之，这是我们的起点：

```solidity
// src/lib/Tick.sol
library Tick {
    struct Info {
        bool initialized;
        uint128 liquidity;
    }
    ...
}

// src/lib/Position.sol
library Position {
    struct Info {
        uint128 liquidity;
    }
    ...
}

// src/UniswapV3Pool.sol
contract UniswapV3Pool {
    using Tick for mapping(int24 => Tick.Info);
    using Position for mapping(bytes32 => Position.Info);
    using Position for Position.Info;

    int24 internal constant MIN_TICK = -887272;
    int24 internal constant MAX_TICK = -MIN_TICK;

    // Pool tokens, immutable
    address public immutable token0;
    address public immutable token1;

    // Packing variables that are read together
    struct Slot0 {
        // Current sqrt(P)
        uint160 sqrtPriceX96;
        // Current tick
        int24 tick;
    }
    Slot0 public slot0;

    // Amount of liquidity, L.
    uint128 public liquidity;

    // Ticks info
    mapping(int24 => Tick.Info) public ticks;
    // Positions info
    mapping(bytes32 => Position.Info) public positions;

    ...
```

`Tick` 和 `Position` 是Uniswap V3 使用的常见两个助手合约。`using A for B` Solidity允许你使用 A 拓展 B。这样可以简化复杂的数据结构。


> 为了简洁起见，我将省略对 Solidity 语法和特性的详细解释。Solidity 拥有非常完善的[文档](https://docs.soliditylang.org/en/latest/)，如果遇到任何不清楚的地方，请随时查阅！

然后在构造器里初始化一些变量:

```solidity
    constructor(
        address token0_,
        address token1_,
        uint160 sqrtPriceX96,
        int24 tick
    ) {
        token0 = token0_;
        token1 = token1_;

        slot0 = Slot0({sqrtPriceX96: sqrtPriceX96, tick: tick});
    }
}
```
在这里，我们将代币地址设置为不可更改，并设置当前价格和tick值——我们不需要为后者提供流动性。

这是我们的起点，本章的目标是使用预先计算和硬编码的值进行第一次交换。

## 铸造

在 Uniswap V2 中提供流动性的过程称为“铸币”。这是因为 V2 合约池会铸币（LP 代币）来换取流动性。V3 虽然不再采用这种方式，但仍然沿用了相同的名称。我们也沿用这个名称：

```solidity
function mint(
    address owner,
    int24 lowerTick,
    int24 upperTick,
    uint128 amount
) external returns (uint256 amount0, uint256 amount1) {
    ...
```

 `mint` 函数的输入:
1. 流动性所有者的地址，用于追踪流动性的所有者。 
1. 价格上下限，用于设定价格区间。 
1. 我们希望提供的流动性金额。 

请注意，用户指定的是 L 值，而不是实际的代币数量。这当然不太方便，但请记住，Pool 合约是一个核心合约——它并非旨在提供用户友好的体验，因为它只应实现核心逻辑。在后面的章节中，我们将创建一个辅助合约，在调用 Pool.mint 之前将代币数量转换为 L 值。

让我们简要概述一如何铸币：
1. 用户指定价格区间和需要的流动性；
1. 合约更新 tick 和头寸的map;
1. 合约计算用户需要发送的代币数量（我们事先计算好，然后写死）
1. 合约收取用户的代币，然后校对发送的数量。


检查一下tick:
```solidity
if (
    lowerTick >= upperTick ||
    lowerTick < MIN_TICK ||
    upperTick > MAX_TICK
) revert InvalidTickRange();
```
确保提供了流动性：
```solidity
if (amount == 0) revert ZeroLiquidity();
```

给一个位置加tick:
```solidity
ticks.update(lowerTick, amount);
ticks.update(upperTick, amount);

Position.Info storage position = positions.get(
    owner,
    lowerTick,
    upperTick
);
position.update(amount);
```

`ticks.update` 函数:

```solidity
// src/lib/Tick.sol
function update(
    mapping(int24 => Tick.Info) storage self,
    int24 tick,
    uint128 liquidityDelta
) internal {
    Tick.Info storage tickInfo = self[tick];
    uint128 liquidityBefore = tickInfo.liquidity;
    uint128 liquidityAfter = liquidityBefore + liquidityDelta;

    if (liquidityBefore == 0) {
        tickInfo.initialized = true;
    }

    tickInfo.liquidity = liquidityAfter;
}
```

如果某个tick的流动性为零，则该函数会初始化该tick，并为其添加新的流动性。如您所见，我们在价格最高和最低tick上都调用了此函数，因此两者都增加了流动性。


`position.update` 函数:
```solidity
// src/libs/Position.sol
function update(Info storage self, uint128 liquidityDelta) internal {
    uint128 liquidityBefore = self.liquidity;
    uint128 liquidityAfter = liquidityBefore + liquidityDelta;

    self.liquidity = liquidityAfter;
}
```

和tick更新函数很像，为指定头寸增加了流动性，我们调用

```solidity
// src/libs/Position.sol
...
function get(
    mapping(bytes32 => Info) storage self,
    address owner,
    int24 lowerTick,
    int24 upperTick
) internal view returns (Position.Info storage position) {
    position = self[
        keccak256(abi.encodePacked(owner, lowerTick, upperTick))
    ];
}
...
```
以获得位置

每个位置都由三个键唯一标识：所有者地址、下限刻度索引和上限刻度索引。我们对这三个进行哈希处理，以降低数据存储成本：哈希处理后，每个key将占用 32 字节，而不是像 owner、lowerTick 和 upperTick 是单独的key时占用 96 字节。

>如果使用三个key，就需要三个map。每个key都会单独存储，并且占用 32 字节，因为（未应用打包时） Solidity 将值存储在 32 字节的槽中。

接下来，继续进行铸币，我们需要计算用户必须存入的金额。幸运的是，我们在上一部分已经推导出了公式并计算出了精确的金额。因此，我们将直接将它们硬编码到代码中：

```solidity
amount0 = 0.998976618347425280 ether;
amount1 = 5000 ether;
```

> 我们在后面没的章节中把这些提换为实际计算结果。

根据增加的 amount 来更新流动性
We will also update the `liquidity` of the pool, based on the `amount` being added.

```solidity
liquidity += uint128(amount);
```
现在可以把用户的代币划走，我们可以使用回调：
```solidity
function mint(...) ... {
    ...

    uint256 balance0Before;
    uint256 balance1Before;
    if (amount0 > 0) balance0Before = balance0();
    if (amount1 > 0) balance1Before = balance1();
    IUniswapV3MintCallback(msg.sender).uniswapV3MintCallback(
        amount0,
        amount1
    );
    if (amount0 > 0 && balance0Before + amount0 > balance0())
        revert InsufficientInputAmount();
    if (amount1 > 0 && balance1Before + amount1 > balance1())
        revert InsufficientInputAmount();

    ...
}

function balance0() internal returns (uint256 balance) {
    balance = IERC20(token0).balanceOf(address(this));
}

function balance1() internal returns (uint256 balance) {
    balance = IERC20(token1).balanceOf(address(this));
}
```
首先，我们记录当前的代币余额。然后，我们调用调用方的 uniswapV3MintCallback 方法——这就是回调函数。一般调用者（调用 mint 的任何人）都是合约，因为非合约地址无法在以太坊中实现功能。这里使用回调函数虽然一点也不方便用户使用，但可以让合约根据其当前状态计算代币数量——这至关重要，因为我们不能信任用户。

调用者需要实现 uniswapV3MintCallback 函数，并将代币转移到 Pool 合约。调用回调函数后，我们会继续检查 Pool 合约的余额是否发生变化：我们要求余额至少分别增加 amount0 和 amount1——这意味着调用者已将代币转移到池中。
The caller is expected to implement `uniswapV3MintCallback` and transfer tokens to the Pool contract in this function.  After calling the callback function, we continue with checking whether the Pool contract balances have changed or not: we require them to increase by at least `amount0` and `amount1` respectively–this would mean the caller has transferred tokens to the pool.

最后触发一个 `Mint` 事件:
```solidity
emit Mint(msg.sender, owner, lowerTick, upperTick, amount, amount0, amount1);
```

事件是以太坊中合约数据被索引以便后续搜索的方式。最佳实践是在合约状态发生改变时触发一个事件，以便区块链浏览器了解这一变化。事件也包含有用的信息。在我们的案例中，这些信息包括调用者的地址、流动性持有者的地址、价格上下限、新增流动性以及代币数量。该信息将以日志的形式存储，任何其他人都可以收集所有合约事件并重现合约活动，而无需遍历和分析所有区块和交易。

搞定了！呼！现在，让我们测试一下铸币功能。

## 测试

我们现在还不知道功能对不对，在部署到其他地方之前，我们需要写一堆这事来保证合约正常工作。很幸运，Forge 是一个很棒的测试框架，测试会非常容易。

建一个测试文件:
```solidity
// test/UniswapV3Pool.t.sol
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.14;

import "forge-std/Test.sol";

contract UniswapV3PoolTest is Test {
    function setUp() public {}

    function testExample() public {
        assertTrue(true);
    }
}
```

Let's run it:
```shell
$ forge test
Running 1 test for test/UniswapV3Pool.t.sol:UniswapV3PoolTest
[PASS] testExample() (gas: 279)
Test result: ok. 1 passed; 0 failed; finished in 5.07ms
```
通过了，当然现在我们的测试用例之判断 true 等不等于 true。

测试合约是从`forge-std/Test.sol`继承的合约。这个合约是一系列的测试工具，我们将逐步熟悉它们。如果您不想等待，可以打开 `lib/forge-std/src/Test.sol` 快速浏览一遍。

测试合约遵循以下规范：
1. setUp 函数用于设置测试用例。在每个测试用例中，我们需要一个配置好的环境，例如已部署的合约、已铸造的代币和已初始化的资金池——所有这些都将在 setUp 函数中完成。
1. 每个测试用例都以 test 前缀开头，例如 testMint()。这样 Forge 就可以将测试用例与辅助函数区分开来（我们也可以创建任何我们想要的函数）。

来好好测试以下铸币功能.

### 测试代币

为了测试铸币功能，我们需要代币。这不成问题，因为我们可以在测试中部署任何合约！此外，Forge 可以将开源合约作为依赖项安装。具体来说，我们需要一个具有铸币功能的 ERC20 合约。我们将使用 [Solmate](https://github.com/Rari-Capital/solmate) 中的 ERC20 合约（Solmate 是一系列 gas 优化合约的集合），并创建一个继承自 Solmate 合约并公开铸币的 ERC20 合约（默认情况下是公开的）。

安装 `solmate`:
```shell
$ forge install rari-capital/solmate
```

接下来，我们在测试文件夹中创建 ERC20Mintable.sol 合约（我们只会在测试中使用该合约）：
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.14;

import "solmate/tokens/ERC20.sol";

contract ERC20Mintable is ERC20 {
    constructor(
        string memory _name,
        string memory _symbol,
        uint8 _decimals
    ) ERC20(_name, _symbol, _decimals) {}

    function mint(address to, uint256 amount) public {
        _mint(to, amount);
    }
}
```
我们的 ERC20Mintable 继承了 solmate/tokens/ERC20.sol 的所有功能，并且我们还实现了公开铸造方法，这将允许我们铸造任意数量的代币。

### 铸币

现在我准备好测试铸币。

首先部署所有需要的合约:
```solidity
// test/UniswapV3Pool.t.sol
...
import "./ERC20Mintable.sol";
import "../src/UniswapV3Pool.sol";

contract UniswapV3PoolTest is Test {
    ERC20Mintable token0;
    ERC20Mintable token1;
    UniswapV3Pool pool;

    function setUp() public {
        token0 = new ERC20Mintable("Ether", "ETH", 18);
        token1 = new ERC20Mintable("USDC", "USDC", 18);
    }

    ...
```
在 setUp 函数中，我们部署了代币，而不是合约池！这是因为我们所有的测试用例都将使用相同的代币，但每个用例都将拥有一个唯一的池。

为了使资金池的设置更清晰、更简单，我们将使用单独的函数 setupTestCase 来完成此操作，该函数接受一组测试用例参数。在第一个测试用例中，我们将测试流动性铸造是否成功。以下是测试用例参数的示例：
```solidity
function testMintSuccess() public {
    TestCaseParams memory params = TestCaseParams({
        wethBalance: 1 ether,
        usdcBalance: 5000 ether,
        currentTick: 85176,
        lowerTick: 84222,
        upperTick: 86129,
        liquidity: 1517882343751509868544,
        currentSqrtP: 5602277097478614198912276234240,
        shouldTransferInCallback: true,
        mintLiqudity: true
    });
```
1. 我们计划向资金池存入 1 个 ETH 和 5000 个 USDC。
1. 我们希望当前 tick 为 85176，下限 tick 和上限 tick 分别为 84222 和 86129（这些值已在上一章计算得出）。
1. 我们指定了预先计算的流动性和当前 $\sqrt{P}$ 值。
1. 我们还希望存入流动性（mintLiquidity 参数），并在合约池请求时转移代币（shouldTransferInCallback）。我们不希望在每个测试用例中都执行这些操作，因此需要设置标志位。

接下来，我们使用上述参数调用 setupTestCase：

```solidity
function setupTestCase(TestCaseParams memory params)
    internal
    returns (uint256 poolBalance0, uint256 poolBalance1)
{
    token0.mint(address(this), params.wethBalance);
    token1.mint(address(this), params.usdcBalance);

    pool = new UniswapV3Pool(
        address(token0),
        address(token1),
        params.currentSqrtP,
        params.currentTick
    );

    if (params.mintLiqudity) {
        (poolBalance0, poolBalance1) = pool.mint(
            address(this),
            params.lowerTick,
            params.upperTick,
            params.liquidity
        );
    }

    shouldTransferInCallback = params.shouldTransferInCallback;
}
```

在这个函数中，我们会铸造代币并部署一个流动性池。此外，当设置了 mintLiquidity 标志时，我们会向流动性池中铸造流动性。最后，我们会设置 shouldTransferInCallback 标志，以便在铸造回调函数中读取它：
```solidity
function uniswapV3MintCallback(uint256 amount0, uint256 amount1) public {
    if (shouldTransferInCallback) {
        token0.transfer(msg.sender, amount0);
        token1.transfer(msg.sender, amount1);
    }
}
```
这是一个测试合约，它将提供流动性并调用资金池的铸币函数，目前没有用户。测试合约将扮演用户的角色，因此它可以实现铸币回调函数。

像这样设置测试用例并非强制性的，您可以根据自己的习惯选择最方便的方式。测试合约也只是合约而已。

在 testMintSuccess 中，我们希望确保资金池合约：
1. 从我们这里收取正确数量的代币； 
2. 使用正确的密钥和流动性创建仓位； 
3. 初始化我们指定的上下限价格； 
4. 具有正确的 $\sqrt{P}$​ 和 L。

铸币操作在 setupTestCase 函数中已经完成，所以我们不需要再次执行此操作。该函数还会返回我们提供的数量，因此让我们检查一下：

```solidity
(uint256 poolBalance0, uint256 poolBalance1) = setupTestCase(params);

uint256 expectedAmount0 = 0.998976618347425280 ether;
uint256 expectedAmount1 = 5000 ether;
assertEq(
    poolBalance0,
    expectedAmount0,
    "incorrect token0 deposited amount"
);
assertEq(
    poolBalance1,
    expectedAmount1,
    "incorrect token1 deposited amount"
);
```
我们预期会收到预先计算好的具体金额。我们还可以核实这些金额是否已转入资金池：
```solidity
assertEq(token0.balanceOf(address(pool)), expectedAmount0);
assertEq(token1.balanceOf(address(pool)), expectedAmount1);
```
接下来，我们需要检查资金池为我们创建的仓位。还记得仓位映射中的键是一个哈希值吗？我们需要手动计算它，然后从合约中获取我们的仓位：
```solidity
bytes32 positionKey = keccak256(
    abi.encodePacked(address(this), params.lowerTick, params.upperTick)
);
uint128 posLiquidity = pool.positions(positionKey);
assertEq(posLiquidity, params.liquidity);
```
> 由于 Position.Info 是一个结构体，因此在获取时会被解构：每个字段都会被分配给一个单独的变量。

接下来是tick。同样，方法也很简单：
```solidity
(bool tickInitialized, uint128 tickLiquidity) = pool.ticks(
    params.lowerTick
);
assertTrue(tickInitialized);
assertEq(tickLiquidity, params.liquidity);

(tickInitialized, tickLiquidity) = pool.ticks(params.upperTick);
assertTrue(tickInitialized);
assertEq(tickLiquidity, params.liquidity);
```

And finally, $\sqrt{P}$ and $L$:
```solidity
(uint160 sqrtPriceX96, int24 tick) = pool.slot0();
assertEq(
    sqrtPriceX96,
    5602277097478614198912276234240,m
    "invalid current sqrtP"
);
assertEq(tick, 85176, "invalid current tick");
assertEq(
    pool.liquidity(),
    1517882343751509868544,
    "invalid current liquidity"
);
```

As you can see, writing tests in Solidity is not hard!

### 失败

当然，仅仅测试成功场景是不够的。我们还需要测试失败案例。提供流动性时可能会出现哪些问题？以下是一些提示：
1. 价格上下限过大或过小。 
2. 流动性为零。 
3. 流动性提供者持有的代币不足。

这些场景的实现就交给你们了！欢迎查看[代码仓库](https://github.com/Jeiwan/uniswapv3-code/blob/milestone_1/test/UniswapV3Pool.t.sol)中的代码。