# Uniswap V2 Flash Swap

对应英文原页：https://solidity-by-example.org/defi/uniswap-v2-flash-swap

### Uniswap V2 Flash Swap 示例

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

interface IUniswapV2Callee {
    function uniswapV2Call(
        address sender,
        uint256 amount0,
        uint256 amount1,
        bytes calldata data
    ) external;
}

contract UniswapV2FlashSwap is IUniswapV2Callee {
    address private constant UNISWAP_V2_FACTORY =
        0x5C69bEe701ef814a2B6a3EDD4B1652CB9cc5aA6f;

    address private constant DAI = 0x6B175474E89094C44Da98b954EedeAC495271d0F;
    address private constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;

    IUniswapV2Factory private constant factory =
        IUniswapV2Factory(UNISWAP_V2_FACTORY);

    IERC20 private constant weth = IERC20(WETH);

    IUniswapV2Pair private immutable pair;

    // 本示例中，存储需要偿还的数量
    uint256 public amountToRepay;

    constructor() {
        pair = IUniswapV2Pair(factory.getPair(DAI, WETH));
    }

    function flashSwap(uint256 wethAmount) external {
        // 需要传入一些数据才能触发 uniswapV2Call
        bytes memory data = abi.encode(WETH, msg.sender);

        // amount0Out 是 DAI，amount1Out 是 WETH
        pair.swap(0, wethAmount, address(this), data);
    }

    // 该函数由 DAI/WETH 交易对合约调用
    function uniswapV2Call(
        address sender,
        uint256 amount0,
        uint256 amount1,
        bytes calldata data
    ) external {
        require(msg.sender == address(pair), "not pair");
        require(sender == address(this), "not sender");

        (address tokenBorrow, address caller) =
            abi.decode(data, (address, address));

        // 自定义逻辑写在这里。例如套利代码。
        require(tokenBorrow == WETH, "token borrow != WETH");

        // 约 0.3% 手续费，+1 用于向上取整
        uint256 fee = (amount1 * 3) / 997 + 1;
        amountToRepay = amount1 + fee;

        // 从调用者处转入闪电兑换手续费
        weth.transferFrom(caller, address(this), fee);

        // 偿还
        weth.transfer(address(pair), amountToRepay);
    }
}

interface IUniswapV2Pair {
    function swap(
        uint256 amount0Out,
        uint256 amount1Out,
        address to,
        bytes calldata data
    ) external;
}

interface IUniswapV2Factory {
    function getPair(address tokenA, address tokenB)
        external
        view
        returns (address pair);
}

interface IERC20 {
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function transfer(address recipient, uint256 amount)
        external
        returns (bool);
    function allowance(address owner, address spender)
        external
        view
        returns (uint256);
    function approve(address spender, uint256 amount) external returns (bool);
    function transferFrom(address sender, address recipient, uint256 amount)
        external
        returns (bool);
}

interface IWETH is IERC20 {
    function deposit() external payable;
    function withdraw(uint256 amount) external;
}

```

### 使用 Foundry 测试

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

import {Test} from "forge-std/Test.sol";
import "../../../src/defi/uniswap-v2-flash-swap/UniswapV2FlashSwap.sol";

address constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
address constant DAI = 0x6B175474E89094C44Da98b954EedeAC495271d0F;
address constant USDC = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;

contract UniswapV2FlashSwapTest is Test {
    IWETH private weth = IWETH(WETH);

    UniswapV2FlashSwap private uni = new UniswapV2FlashSwap();

    function setUp() public {}

    function testFlashSwap() public {
        weth.deposit{value: 1e18}();
        // 授权闪电兑换手续费
        weth.approve(address(uni), 1e18);

        uint256 amountToBorrow = 10 * 1e18;
        uni.flashSwap(amountToBorrow);

        assertGt(uni.amountToRepay(), amountToBorrow);
    }
}

```

### 链接

[Foundry](https://github.com/foundry-rs/foundry)

[Uniswap V3 Foundry 示例](https://github.com/t4sk/defi-notes)

---
## 关注我们
[Yanbo的Twitter](https://twitter.com/YanboOfficial)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)

加入Web3Club 官方讨论群：YanboTravelAllWorld（微信号）

[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
