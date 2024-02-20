# Uniswap V2 互换
swapExactTokensForTokens 将所有代币出售为另一个代币。

swapTokensForExactTokens 购买调用者设置的特定数量的代币。

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

contract UniswapV2SwapExamples {
    address private constant UNISWAP_V2_ROUTER =
        0x7a250d5630B4cF539739dF2C5dAcb4c659F2488D;

    address private constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address private constant DAI = 0x6B175474E89094C44Da98b954EedeAC495271d0F;
    address constant USDC = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;

    IUniswapV2Router private router = IUniswapV2Router(UNISWAP_V2_ROUTER);
    IERC20 private weth = IERC20(WETH);
    IERC20 private dai = IERC20(DAI);

    // Swap WETH to DAI
    function swapSingleHopExactAmountIn(
        uint amountIn,
        uint amountOutMin
    ) external returns (uint amountOut) {
        weth.transferFrom(msg.sender, address(this), amountIn);
        weth.approve(address(router), amountIn);

        address[] memory path;
        path = new address[](2);
        path[0] = WETH;
        path[1] = DAI;

        uint[] memory amounts = router.swapExactTokensForTokens(
            amountIn,
            amountOutMin,
            path,
            msg.sender,
            block.timestamp
        );

        // amounts[0] = WETH amount, amounts[1] = DAI amount
        return amounts[1];
    }

    // Swap DAI -> WETH -> USDC
    function swapMultiHopExactAmountIn(
        uint amountIn,
        uint amountOutMin
    ) external returns (uint amountOut) {
        dai.transferFrom(msg.sender, address(this), amountIn);
        dai.approve(address(router), amountIn);

        address[] memory path;
        path = new address[](3);
        path[0] = DAI;
        path[1] = WETH;
        path[2] = USDC;

        uint[] memory amounts = router.swapExactTokensForTokens(
            amountIn,
            amountOutMin,
            path,
            msg.sender,
            block.timestamp
        );

        // amounts[0] = DAI amount
        // amounts[1] = WETH amount
        // amounts[2] = USDC amount
        return amounts[2];
    }

    // Swap WETH to DAI
    function swapSingleHopExactAmountOut(
        uint amountOutDesired,
        uint amountInMax
    ) external returns (uint amountOut) {
        weth.transferFrom(msg.sender, address(this), amountInMax);
        weth.approve(address(router), amountInMax);

        address[] memory path;
        path = new address[](2);
        path[0] = WETH;
        path[1] = DAI;

        uint[] memory amounts = router.swapTokensForExactTokens(
            amountOutDesired,
            amountInMax,
            path,
            msg.sender,
            block.timestamp
        );

        // Refund WETH to msg.sender
        if (amounts[0] < amountInMax) {
            weth.transfer(msg.sender, amountInMax - amounts[0]);
        }

        return amounts[1];
    }

    // Swap DAI -> WETH -> USDC
    function swapMultiHopExactAmountOut(
        uint amountOutDesired,
        uint amountInMax
    ) external returns (uint amountOut) {
        dai.transferFrom(msg.sender, address(this), amountInMax);
        dai.approve(address(router), amountInMax);

        address[] memory path;
        path = new address[](3);
        path[0] = DAI;
        path[1] = WETH;
        path[2] = USDC;

        uint[] memory amounts = router.swapTokensForExactTokens(
            amountOutDesired,
            amountInMax,
            path,
            msg.sender,
            block.timestamp
        );

        // Refund DAI to msg.sender
        if (amounts[0] < amountInMax) {
            dai.transfer(msg.sender, amountInMax - amounts[0]);
        }

        return amounts[2];
    }
}

interface IUniswapV2Router {
    function swapExactTokensForTokens(
        uint amountIn,
        uint amountOutMin,
        address[] calldata path,
        address to,
        uint deadline
    ) external returns (uint[] memory amounts);

    function swapTokensForExactTokens(
        uint amountOut,
        uint amountInMax,
        address[] calldata path,
        address to,
        uint deadline
    ) external returns (uint[] memory amounts);
}

interface IERC20 {
    function totalSupply() external view returns (uint);

    function balanceOf(address account) external view returns (uint);

    function transfer(address recipient, uint amount) external returns (bool);

    function allowance(address owner, address spender) external view returns (uint);

    function approve(address spender, uint amount) external returns (bool);

    function transferFrom(
        address sender,
        address recipient,
        uint amount
    ) external returns (bool);

    event Transfer(address indexed from, address indexed to, uint value);
    event Approval(address indexed owner, address indexed spender, uint value);
}

interface IWETH is IERC20 {
    function deposit() external payable;

    function withdraw(uint amount) external;
}
```


## 与 Foundry 进行测试

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

import "forge-std/Test.sol";
import "forge-std/console.sol";

import "../src/UniswapV2SwapExamples.sol";

address constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
address constant DAI = 0x6B175474E89094C44Da98b954EedeAC495271d0F;
address constant USDC = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;

contract UniswapV2SwapExamplesTest is Test {
    IWETH private weth = IWETH(WETH);
    IERC20 private dai = IERC20(DAI);
    IERC20 private usdc = IERC20(USDC);

    UniswapV2SwapExamples private uni = new UniswapV2SwapExamples();

    function setUp() public {}

    // Swap WETH -> DAI
    function testSwapSingleHopExactAmountIn() public {
        uint wethAmount = 1e18;
        weth.deposit{value: wethAmount}();
        weth.approve(address(uni), wethAmount);

        uint daiAmountMin = 1;
        uint daiAmountOut = uni.swapSingleHopExactAmountIn(wethAmount, daiAmountMin);

        console.log("DAI", daiAmountOut);
        assertGe(daiAmountOut, daiAmountMin, "amount out < min");
    }

    // Swap DAI -> WETH -> USDC
    function testSwapMultiHopExactAmountIn() public {
        // Swap WETH -> DAI
        uint wethAmount = 1e18;
        weth.deposit{value: wethAmount}();
        weth.approve(address(uni), wethAmount);

        uint daiAmountMin = 1;
        uni.swapSingleHopExactAmountIn(wethAmount, daiAmountMin);

        // Swap DAI -> WETH -> USDC
        uint daiAmountIn = 1e18;
        dai.approve(address(uni), daiAmountIn);

        uint usdcAmountOutMin = 1;
        uint usdcAmountOut = uni.swapMultiHopExactAmountIn(
            daiAmountIn,
            usdcAmountOutMin
        );

        console.log("USDC", usdcAmountOut);
        assertGe(usdcAmountOut, usdcAmountOutMin, "amount out < min");
    }

    // Swap WETH -> DAI
    function testSwapSingleHopExactAmountOut() public {
        uint wethAmount = 1e18;
        weth.deposit{value: wethAmount}();
        weth.approve(address(uni), wethAmount);

        uint daiAmountDesired = 1e18;
        uint daiAmountOut = uni.swapSingleHopExactAmountOut(
            daiAmountDesired,
            wethAmount
        );

        console.log("DAI", daiAmountOut);
        assertEq(daiAmountOut, daiAmountDesired, "amount out != amount out desired");
    }

    // Swap DAI -> WETH -> USDC
    function testSwapMultiHopExactAmountOut() public {
        // Swap WETH -> DAI
        uint wethAmount = 1e18;
        weth.deposit{value: wethAmount}();
        weth.approve(address(uni), wethAmount);

        // Buy 100 DAI
        uint daiAmountOut = 100 * 1e18;
        uni.swapSingleHopExactAmountOut(daiAmountOut, wethAmount);

        // Swap DAI -> WETH -> USDC
        dai.approve(address(uni), daiAmountOut);

        uint amountOutDesired = 1e6;
        uint amountOut = uni.swapMultiHopExactAmountOut(amountOutDesired, daiAmountOut);

        console.log("USDC", amountOut);
        assertEq(amountOut, amountOutDesired, "amount out != amount out desired");
    }
}
```


