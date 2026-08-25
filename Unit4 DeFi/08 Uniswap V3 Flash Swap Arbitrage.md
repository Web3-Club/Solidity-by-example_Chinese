# Uniswap V3 Flash Swap 套利

对应英文原页：https://solidity-by-example.org/defi/uniswap-v3-flash-swap

### Uniswap V3 Flash Swap 套利示例

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

address constant SWAP_ROUTER_02 = 0x68b3465833fb72A70ecDF485E0e4C7bD8665Fc45;

contract UniswapV3FlashSwap {
    ISwapRouter02 constant router = ISwapRouter02(SWAP_ROUTER_02);

    uint160 private constant MIN_SQRT_RATIO = 4295128739;
    uint160 private constant MAX_SQRT_RATIO =
        1461446703485210103287273052203988822378723970342;

    // DAI / WETH 0.3% 兑换手续费 (2000 DAI / WETH)
    // DAI / WETH 0.05% 兑换手续费 (2100 DAI / WETH)
    // 1. 在 pool0 上进行闪电兑换（收到 WETH）
    // 2. 在 pool1 上兑换（WETH -> DAI）
    // 3. 将 DAI 发送到 pool0
    // profit = 从 pool1 收到的 DAI - 偿还给 pool0 的 DAI

    function flashSwap(
        address pool0,
        uint24 fee1,
        address tokenIn,
        address tokenOut,
        uint256 amountIn
    ) external {
        bool zeroForOne = tokenIn < tokenOut;
        // 0 -> 1 => sqrt 价格下降
        // 1 -> 0 => sqrt 价格上升
        uint160 sqrtPriceLimitX96 =
            zeroForOne ? MIN_SQRT_RATIO + 1 : MAX_SQRT_RATIO - 1;

        bytes memory data = abi.encode(
            msg.sender, pool0, fee1, tokenIn, tokenOut, amountIn, zeroForOne
        );

        IUniswapV3Pool(pool0).swap({
            recipient: address(this),
            zeroForOne: zeroForOne,
            amountSpecified: int256(amountIn),
            sqrtPriceLimitX96: sqrtPriceLimitX96,
            data: data
        });
    }

    function _swap(
        address tokenIn,
        address tokenOut,
        uint24 fee,
        uint256 amountIn,
        uint256 amountOutMin
    ) private returns (uint256 amountOut) {
        IERC20(tokenIn).approve(address(router), amountIn);

        ISwapRouter02.ExactInputSingleParams memory params = ISwapRouter02
            .ExactInputSingleParams({
            tokenIn: tokenIn,
            tokenOut: tokenOut,
            fee: fee,
            recipient: address(this),
            amountIn: amountIn,
            amountOutMinimum: amountOutMin,
            sqrtPriceLimitX96: 0
        });

        amountOut = router.exactInputSingle(params);
    }

    function uniswapV3SwapCallback(
        int256 amount0,
        int256 amount1,
        bytes calldata data
    ) external {
        // 解码 data
        (
            address caller,
            address pool0,
            uint24 fee1,
            address tokenIn,
            address tokenOut,
            uint256 amountIn,
            bool zeroForOne
        ) = abi.decode(
            data, (address, address, uint24, address, address, uint256, bool)
        );

        uint256 amountOut = zeroForOne ? uint256(-amount1) : uint256(-amount0);

        // pool0 -> tokenIn -> tokenOut (amountOut)
        // 在 pool 1 上兑换（tokenOut -> tokenIn）
        uint256 buyBackAmount = _swap({
            tokenIn: tokenOut,
            tokenOut: tokenIn,
            fee: fee1,
            amountIn: amountOut,
            amountOutMin: amountIn
        });

        // 偿还 pool 0
        uint256 profit = buyBackAmount - amountIn;
        require(profit > 0, "profit = 0");

        IERC20(tokenIn).transfer(pool0, amountIn);
        IERC20(tokenIn).transfer(caller, profit);
    }
}

interface ISwapRouter02 {
    struct ExactInputSingleParams {
        address tokenIn;
        address tokenOut;
        uint24 fee;
        address recipient;
        uint256 amountIn;
        uint256 amountOutMinimum;
        uint160 sqrtPriceLimitX96;
    }

    function exactInputSingle(ExactInputSingleParams calldata params)
        external
        payable
        returns (uint256 amountOut);
}

interface IUniswapV3Pool {
    function swap(
        address recipient,
        bool zeroForOne,
        int256 amountSpecified,
        uint160 sqrtPriceLimitX96,
        bytes calldata data
    ) external returns (int256 amount0, int256 amount1);
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
pragma solidity 0.8.26;

import {Test, console2} from "forge-std/Test.sol";
import {
    UniswapV3FlashSwap,
    IUniswapV3Pool,
    ISwapRouter02,
    IERC20,
    IWETH
} from "../../../src/defi/uniswap-v3-flash-swap/UniswapV3FlashSwap.sol";

address constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
address constant DAI = 0x6B175474E89094C44Da98b954EedeAC495271d0F;
address constant SWAP_ROUTER_02 = 0x68b3465833fb72A70ecDF485E0e4C7bD8665Fc45;
address constant DAI_WETH_POOL_3000 = 0xC2e9F25Be6257c210d7Adf0D4Cd6E3E881ba25f8;
address constant DAI_WETH_POOL_500 = 0x60594a405d53811d3BC4766596EFD80fd545A270;
uint24 constant FEE_0 = 3000;
uint24 constant FEE_1 = 500;

contract UniswapV3FlashTest is Test {
    IERC20 private constant dai = IERC20(DAI);
    IWETH private constant weth = IWETH(WETH);
    ISwapRouter02 private constant router = ISwapRouter02(SWAP_ROUTER_02);
    IUniswapV3Pool private constant pool0 = IUniswapV3Pool(DAI_WETH_POOL_3000);
    IUniswapV3Pool private constant pool1 = IUniswapV3Pool(DAI_WETH_POOL_500);
    UniswapV3FlashSwap private flashSwap;

    uint256 private constant DAI_AMOUNT_IN = 10 * 1e18;

    function setUp() public {
        flashSwap = new UniswapV3FlashSwap();

        // 制造套利机会——让 pool0 上的 WETH 更便宜
        weth.deposit{value: 500 * 1e18}();
        weth.approve(address(router), 500 * 1e18);
        router.exactInputSingle(
            ISwapRouter02.ExactInputSingleParams({
                tokenIn: WETH,
                tokenOut: DAI,
                fee: FEE_0,
                recipient: address(0),
                amountIn: 500 * 1e18,
                amountOutMinimum: 0,
                sqrtPriceLimitX96: 0
            })
        );
    }

    function test_flashSwap() public {
        uint256 bal0 = dai.balanceOf(address(this));
        flashSwap.flashSwap({
            pool0: address(pool0),
            fee1: FEE_1,
            tokenIn: DAI,
            tokenOut: WETH,
            amountIn: DAI_AMOUNT_IN
        });
        uint256 bal1 = dai.balanceOf(address(this));
        uint256 profit = bal1 - bal0;
        assertGt(profit, 0, "profit = 0");
        console2.log("Profit %e", profit);
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
