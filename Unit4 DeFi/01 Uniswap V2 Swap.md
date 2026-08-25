# Uniswap V2 Swap

对应英文原页：https://solidity-by-example.org/defi/uniswap-v2

`swapExactTokensForTokens` 将全部代币卖出，换为另一种代币。

`swapTokensForExactTokens` 买入调用者指定数量的代币。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

contract UniswapV2SwapExamples {
    address private constant UNISWAP_V2_ROUTER =
        0x7a250d5630B4cF539739dF2C5dAcb4c659F2488D;

    address private constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address private constant DAI = 0x6B175474E89094C44Da98b954EedeAC495271d0F;
    address constant USDC = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;

    IUniswapV2Router private router = IUniswapV2Router(UNISWAP_V2_ROUTER);
    IERC20 private weth = IERC20(WETH);
    IERC20 private dai = IERC20(DAI);

    // 将 WETH 兑换为 DAI
    function swapSingleHopExactAmountIn(uint256 amountIn, uint256 amountOutMin)
        external
        returns (uint256 amountOut)
    {
        weth.transferFrom(msg.sender, address(this), amountIn);
        weth.approve(address(router), amountIn);

        address[] memory path;
        path = new address[](2);
        path[0] = WETH;
        path[1] = DAI;

        uint256[] memory amounts = router.swapExactTokensForTokens(
            amountIn, amountOutMin, path, msg.sender, block.timestamp
        );

        // amounts[0] = WETH 数量，amounts[1] = DAI 数量
        return amounts[1];
    }

    // 兑换 DAI -> WETH -> USDC
    function swapMultiHopExactAmountIn(uint256 amountIn, uint256 amountOutMin)
        external
        returns (uint256 amountOut)
    {
        dai.transferFrom(msg.sender, address(this), amountIn);
        dai.approve(address(router), amountIn);

        address[] memory path;
        path = new address[](3);
        path[0] = DAI;
        path[1] = WETH;
        path[2] = USDC;

        uint256[] memory amounts = router.swapExactTokensForTokens(
            amountIn, amountOutMin, path, msg.sender, block.timestamp
        );

        // amounts[0] = DAI 数量
        // amounts[1] = WETH 数量
        // amounts[2] = USDC 数量
        return amounts[2];
    }

    // 将 WETH 兑换为 DAI
    function swapSingleHopExactAmountOut(
        uint256 amountOutDesired,
        uint256 amountInMax
    ) external returns (uint256 amountOut) {
        weth.transferFrom(msg.sender, address(this), amountInMax);
        weth.approve(address(router), amountInMax);

        address[] memory path;
        path = new address[](2);
        path[0] = WETH;
        path[1] = DAI;

        uint256[] memory amounts = router.swapTokensForExactTokens(
            amountOutDesired, amountInMax, path, msg.sender, block.timestamp
        );

        // 将剩余 WETH 退还给 msg.sender
        if (amounts[0] < amountInMax) {
            weth.transfer(msg.sender, amountInMax - amounts[0]);
        }

        return amounts[1];
    }

    // 兑换 DAI -> WETH -> USDC
    function swapMultiHopExactAmountOut(
        uint256 amountOutDesired,
        uint256 amountInMax
    ) external returns (uint256 amountOut) {
        dai.transferFrom(msg.sender, address(this), amountInMax);
        dai.approve(address(router), amountInMax);

        address[] memory path;
        path = new address[](3);
        path[0] = DAI;
        path[1] = WETH;
        path[2] = USDC;

        uint256[] memory amounts = router.swapTokensForExactTokens(
            amountOutDesired, amountInMax, path, msg.sender, block.timestamp
        );

        // 将剩余 DAI 退还给 msg.sender
        if (amounts[0] < amountInMax) {
            dai.transfer(msg.sender, amountInMax - amounts[0]);
        }

        return amounts[2];
    }
}

interface IUniswapV2Router {
    function swapExactTokensForTokens(
        uint256 amountIn,
        uint256 amountOutMin,
        address[] calldata path,
        address to,
        uint256 deadline
    ) external returns (uint256[] memory amounts);

    function swapTokensForExactTokens(
        uint256 amountOut,
        uint256 amountInMax,
        address[] calldata path,
        address to,
        uint256 deadline
    ) external returns (uint256[] memory amounts);
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

import {Test, console2} from "forge-std/Test.sol";
import {
    UniswapV2SwapExamples,
    IERC20,
    IWETH
} from "../../../src/defi/uniswap-v2/UniswapV2SwapExamples.sol";

address constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
address constant DAI = 0x6B175474E89094C44Da98b954EedeAC495271d0F;
address constant USDC = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;

contract UniswapV2SwapExamplesTest is Test {
    IWETH private weth = IWETH(WETH);
    IERC20 private dai = IERC20(DAI);
    IERC20 private usdc = IERC20(USDC);

    UniswapV2SwapExamples private uni = new UniswapV2SwapExamples();

    function setUp() public {}

    // 兑换 WETH -> DAI
    function testSwapSingleHopExactAmountIn() public {
        uint256 wethAmount = 1e18;
        weth.deposit{value: wethAmount}();
        weth.approve(address(uni), wethAmount);

        uint256 daiAmountMin = 1;
        uint256 daiAmountOut =
            uni.swapSingleHopExactAmountIn(wethAmount, daiAmountMin);

        console2.log("DAI", daiAmountOut);
        assertGe(daiAmountOut, daiAmountMin, "amount out < min");
    }

    // 兑换 DAI -> WETH -> USDC
    function testSwapMultiHopExactAmountIn() public {
        // 兑换 WETH -> DAI
        uint256 wethAmount = 1e18;
        weth.deposit{value: wethAmount}();
        weth.approve(address(uni), wethAmount);

        uint256 daiAmountMin = 1;
        uni.swapSingleHopExactAmountIn(wethAmount, daiAmountMin);

        // 兑换 DAI -> WETH -> USDC
        uint256 daiAmountIn = 1e18;
        dai.approve(address(uni), daiAmountIn);

        uint256 usdcAmountOutMin = 1;
        uint256 usdcAmountOut =
            uni.swapMultiHopExactAmountIn(daiAmountIn, usdcAmountOutMin);

        console2.log("USDC", usdcAmountOut);
        assertGe(usdcAmountOut, usdcAmountOutMin, "amount out < min");
    }

    // 兑换 WETH -> DAI
    function testSwapSingleHopExactAmountOut() public {
        uint256 wethAmount = 1e18;
        weth.deposit{value: wethAmount}();
        weth.approve(address(uni), wethAmount);

        uint256 daiAmountDesired = 1e18;
        uint256 daiAmountOut =
            uni.swapSingleHopExactAmountOut(daiAmountDesired, wethAmount);

        console2.log("DAI", daiAmountOut);
        assertEq(
            daiAmountOut, daiAmountDesired, "amount out != amount out desired"
        );
    }

    // 兑换 DAI -> WETH -> USDC
    function testSwapMultiHopExactAmountOut() public {
        // 兑换 WETH -> DAI
        uint256 wethAmount = 1e18;
        weth.deposit{value: wethAmount}();
        weth.approve(address(uni), wethAmount);

        // 买入 100 DAI
        uint256 daiAmountOut = 100 * 1e18;
        uni.swapSingleHopExactAmountOut(daiAmountOut, wethAmount);

        // 兑换 DAI -> WETH -> USDC
        dai.approve(address(uni), daiAmountOut);

        uint256 amountOutDesired = 1e6;
        uint256 amountOut =
            uni.swapMultiHopExactAmountOut(amountOutDesired, daiAmountOut);

        console2.log("USDC", amountOut);
        assertEq(
            amountOut, amountOutDesired, "amount out != amount out desired"
        );
    }
}

```

---
## 关注我们
[Yanbo的Twitter](https://twitter.com/YanboOfficial)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)

加入Web3Club 官方讨论群：YanboTravelAllWorld（微信号）

[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
