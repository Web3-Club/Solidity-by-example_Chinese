# Uniswap V3 交换

Uniswap V3 交换示例，包括单跳交换、多跳交换（ExactInput/ExactOutput），以及使用 SwapRouter 和 SwapRouter02 的不同方式。

```solidity
// SPDX-License-Identifier: MIT  // SPDX 许可证标识符：MIT 许可证
pragma solidity ^0.8.26;  // 版本指令，要求使用 Solidity 0.8.26 及以上版本

// ===== SwapRouter02 版本（推荐）=====

address constant SWAP_ROUTER_02 = 0x68b3465833fb72A70ecDF485E0e4C7bD8665Fc45;  // 通用路由器
address constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
address constant DAI = 0x6B175474E89094C44Da98b954EedeAC495271d0F;
address constant USDC = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;

// UniswapV3SingleHopSwap 合约：单跳交换（SwapRouter02）
contract UniswapV3SingleHopSwap {  // 定义单跳交换合约
    ISwapRouter02 private constant router = ISwapRouter02(SWAP_ROUTER_02);
    IERC20 private constant weth = IERC20(WETH);
    IERC20 private constant dai = IERC20(DAI);

    // swapExactInputSingleHop 函数：精确输入的单跳交换
    function swapExactInputSingleHop(uint256 amountIn, uint256 amountOutMin)
        external
    {
        weth.transferFrom(msg.sender, address(this), amountIn);  // 转入输入代币
        weth.approve(address(router), amountIn);                 // 授权路由器

        ISwapRouter02.ExactInputSingleParams memory params = ISwapRouter02
            .ExactInputSingleParams({
            tokenIn: WETH,           // 输入代币
            tokenOut: DAI,           // 输出代币
            fee: 3000,               // 费率等级（3000 = 0.3%）
            recipient: msg.sender,   // 输出代币接收者
            amountIn: amountIn,      // 精确输入量
            amountOutMinimum: amountOutMin,  // 最小输出量（滑点保护）
            sqrtPriceLimitX96: 0     // 价格限制（0 = 无限制）
        });

        router.exactInputSingle(params);
    }

    // swapExactOutputSingleHop 函数：精确输出的单跳交换
    function swapExactOutputSingleHop(uint256 amountOut, uint256 amountInMax)
        external
    {
        weth.transferFrom(msg.sender, address(this), amountInMax);
        weth.approve(address(router), amountInMax);

        ISwapRouter02.ExactOutputSingleParams memory params = ISwapRouter02
            .ExactOutputSingleParams({
            tokenIn: WETH,
            tokenOut: DAI,
            fee: 3000,
            recipient: msg.sender,
            amountOut: amountOut,         // 精确输出量
            amountInMaximum: amountInMax, // 最大输入量（滑点保护）
            sqrtPriceLimitX96: 0
        });

        uint256 amountIn = router.exactOutputSingle(params);

        // 退还多转入的代币
        if (amountIn < amountInMax) {
            weth.approve(address(router), 0);
            weth.transfer(msg.sender, amountInMax - amountIn);
        }
    }
}

// UniswapV3MultiHopSwap 合约：多跳交换（通过中间代币路由）
contract UniswapV3MultiHopSwap {  // 定义多跳交换合约
    ISwapRouter02 private constant router = ISwapRouter02(SWAP_ROUTER_02);
    IERC20 private constant weth = IERC20(WETH);
    IERC20 private constant dai = IERC20(DAI);

    // swapExactInputMultiHop 函数：精确输入的多跳交换（WETH → USDC → DAI）
    function swapExactInputMultiHop(uint256 amountIn, uint256 amountOutMin)
        external
    {
        weth.transferFrom(msg.sender, address(this), amountIn);
        weth.approve(address(router), amountIn);

        // Path 编码格式: [token0, fee1, token1, fee2, token2, ...]
        bytes memory path =
            abi.encodePacked(WETH, uint24(3000), USDC, uint24(100), DAI);

        ISwapRouter02.ExactInputParams memory params = ISwapRouter02
            .ExactInputParams({
            path: path,
            recipient: msg.sender,
            amountIn: amountIn,
            amountOutMinimum: amountOutMin
        });

        router.exactInput(params);
    }

    // swapExactOutputMultiHop 函数：精确输出的多跳交换（DAI → USDC → WETH）
    function swapExactOutputMultiHop(uint256 amountOut, uint256 amountInMax)
        external
    {
        weth.transferFrom(msg.sender, address(this), amountInMax);
        weth.approve(address(router), amountInMax);

        bytes memory path =
            abi.encodePacked(DAI, uint24(100), USDC, uint24(3000), WETH);

        ISwapRouter02.ExactOutputParams memory params = ISwapRouter02
            .ExactOutputParams({
            path: path,
            recipient: msg.sender,
            amountOut: amountOut,
            amountInMaximum: amountInMax
        });

        uint256 amountIn = router.exactOutput(params);

        // 退还多余输入
        if (amountIn < amountInMax) {
            weth.approve(address(router), 0);
            weth.transfer(msg.sender, amountInMax - amountIn);
        }
    }
}

// ===== SwapRouter (V1) 版本 =====

// UniswapV3SwapExamples 合约：旧版路由器示例
contract UniswapV3SwapExamples {  // 定义旧版交换示例
    ISwapRouter constant router = ISwapRouter(0xE592427A0AEce92De3Edee1F18E0157C05861564);

    function swapExactInputSingleHop(address tokenIn, address tokenOut, uint24 poolFee, uint256 amountIn)
        external returns (uint256 amountOut)
    {
        IERC20(tokenIn).transferFrom(msg.sender, address(this), amountIn);
        IERC20(tokenIn).approve(address(router), amountIn);
        // ...调用 router.exactInputSingle(params)
    }

    function swapExactInputMultiHop(bytes calldata path, address tokenIn, uint256 amountIn)
        external returns (uint256 amountOut)
    {
        IERC20(tokenIn).transferFrom(msg.sender, address(this), amountIn);
        IERC20(tokenIn).approve(address(router), amountIn);
        // ...调用 router.exactInput(params)
    }
}

// ===== 接口定义 =====

interface ISwapRouter02 {
    struct ExactInputSingleParams {
        address tokenIn; address tokenOut; uint24 fee; address recipient;
        uint256 amountIn; uint256 amountOutMinimum; uint160 sqrtPriceLimitX96;
    }
    function exactInputSingle(ExactInputSingleParams calldata params) external payable returns (uint256 amountOut);

    struct ExactOutputSingleParams {
        address tokenIn; address tokenOut; uint24 fee; address recipient;
        uint256 amountOut; uint256 amountInMaximum; uint160 sqrtPriceLimitX96;
    }
    function exactOutputSingle(ExactOutputSingleParams calldata params) external payable returns (uint256 amountIn);

    struct ExactInputParams {
        bytes path; address recipient; uint256 amountIn; uint256 amountOutMinimum;
    }
    function exactInput(ExactInputParams calldata params) external payable returns (uint256 amountOut);

    struct ExactOutputParams {
        bytes path; address recipient; uint256 amountOut; uint256 amountInMaximum;
    }
    function exactOutput(ExactOutputParams calldata params) external payable returns (uint256 amountIn);
}

interface ISwapRouter {
    function exactInputSingle(ExactInputSingleParams calldata) external payable returns (uint256);
    function exactInput(ExactInputParams calldata) external payable returns (uint256);
}

interface IERC20 { /* 标准 ERC20 接口 */ }
interface IWETH is IERC20 {
    function deposit() external payable;
    function withdraw(uint256 amount) external;
}
```

## Path 编码格式

多跳交换的路径编码方式：

```
[token0(20B)] [fee1(3B)] [token1(20B)] [fee2(3B)] [token2(20B)] ...
```

每个步骤为 `(token + fee)`，最终以输出代币结尾，解码后得到完整的交换路径。

## V3 费率等级

| 费率 | 值 | 适用场景 |
|------|-----|----------|
| 0.01% | 100 | 稳定币对 |
| 0.05% | 500 | 稳定币/蓝筹资产 |
| 0.30% | 3000 | 标准交易对 |
| 1.00% | 10000 | 波动性资产 |
