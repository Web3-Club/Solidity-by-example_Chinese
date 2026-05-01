# Uniswap V3 闪电交换

Uniswap V3 闪电交换（Flash Swap）利用不同费率等级池之间的价格差进行套利。在池 0 借出 WETH → 在池 1 将 WETH 换回 DAI → 归还池 0 + 赚取差价。

```solidity
// SPDX-License-Identifier: MIT  // SPDX 许可证标识符：MIT 许可证
pragma solidity ^0.8.26;  // 版本指令，要求使用 Solidity 0.8.26 及以上版本

address constant SWAP_ROUTER_02 = 0x68b3465833fb72A70ecDF485E0e4C7bD8665Fc45;

// UniswapV3FlashSwap 合约：利用 V3 不同费率池进行套利
contract UniswapV3FlashSwap {  // 定义闪电交换套利合约
    ISwapRouter02 constant router = ISwapRouter02(SWAP_ROUTER_02);

    uint160 private constant MIN_SQRT_RATIO = 4295128739;
    uint160 private constant MAX_SQRT_RATIO =
        1461446703485210103287273052203988822378723970342;

    // 套利逻辑：
    // 1. 在 pool0 借出 WETH（闪电交换）
    // 2. 在 pool1 将 WETH 换为 DAI
    // 3. 将 DAI 归还 pool0
    // 利润 = pool1 换得的 DAI - pool0 需归还的 DAI

    function flashSwap(
        address pool0,
        uint24 fee1,
        address tokenIn,
        address tokenOut,
        uint256 amountIn
    ) external {
        bool zeroForOne = tokenIn < tokenOut;
        // zeroForOne: 0→1（价格平方根下降）
        // !zeroForOne: 1→0（价格平方根上升）
        uint160 sqrtPriceLimitX96 =
            zeroForOne ? MIN_SQRT_RATIO + 1 : MAX_SQRT_RATIO - 1;

        bytes memory data = abi.encode(
            msg.sender, pool0, fee1, tokenIn, tokenOut, amountIn, zeroForOne
        );

        // 在 pool0 发起闪电交换
        IUniswapV3Pool(pool0).swap({
            recipient: address(this),
            zeroForOne: zeroForOne,
            amountSpecified: int256(amountIn),  // 正数 = 精确输入（借入 tokenIn）
            sqrtPriceLimitX96: sqrtPriceLimitX96,
            data: data
        });
    }

    // _swap 内部函数：通过 SwapRouter02 在 pool1 执行交换
    function _swap(
        address tokenIn, address tokenOut,
        uint24 fee, uint256 amountIn, uint256 amountOutMin
    ) private returns (uint256 amountOut) {
        IERC20(tokenIn).approve(address(router), amountIn);
        ISwapRouter02.ExactInputSingleParams memory params = ISwapRouter02
            .ExactInputSingleParams({
            tokenIn: tokenIn, tokenOut: tokenOut, fee: fee,
            recipient: address(this), amountIn: amountIn,
            amountOutMinimum: amountOutMin, sqrtPriceLimitX96: 0
        });
        amountOut = router.exactInputSingle(params);
    }

    // uniswapV3SwapCallback 函数：闪电交换回调
    function uniswapV3SwapCallback(
        int256 amount0, int256 amount1, bytes calldata data
    ) external {
        // 解码回调数据
        (address caller, address pool0, uint24 fee1,
         address tokenIn, address tokenOut,
         uint256 amountIn, bool zeroForOne
        ) = abi.decode(
            data, (address, address, uint24, address, address, uint256, bool)
        );

        uint256 amountOut = zeroForOne ? uint256(-amount1) : uint256(-amount0);

        // 在 pool1 将借出的代币换回
        uint256 buyBackAmount = _swap({
            tokenIn: tokenOut,
            tokenOut: tokenIn,
            fee: fee1,
            amountIn: amountOut,
            amountOutMin: amountIn  // 至少换回借入量
        });

        // 归还 pool0
        uint256 profit = buyBackAmount - amountIn;
        require(profit > 0, "profit = 0");

        IERC20(tokenIn).transfer(pool0, amountIn);    // 归还本金
        IERC20(tokenIn).transfer(caller, profit);      // 利润转给调用者
    }
}

interface ISwapRouter02 {
    struct ExactInputSingleParams {
        address tokenIn; address tokenOut; uint24 fee; address recipient;
        uint256 amountIn; uint256 amountOutMinimum; uint160 sqrtPriceLimitX96;
    }
    function exactInputSingle(ExactInputSingleParams calldata params) external payable returns (uint256);
}

interface IUniswapV3Pool {
    function swap(address recipient, bool zeroForOne, int256 amountSpecified,
        uint160 sqrtPriceLimitX96, bytes calldata data)
        external returns (int256 amount0, int256 amount1);
}

interface IERC20 { /* 标准 ERC20 接口 */ }
```

## 套利流程

```
pool0 (0.3% fee)           pool1 (0.05% fee)
    |------------ flash swap --------->|
    借出 WETH                   将 WETH 换为 DAI
    |                                    |
    归还 DAI    <------------   DAI 换回
    |                                    |
   利润 = 换回 DAI - 归还 DAI
```

利用不同费率池的价格差异，可以实现无本金套利。
