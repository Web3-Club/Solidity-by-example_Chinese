# Uniswap V4 交换

Uniswap V4 使用单例 PoolManager 配合 unlock/unlockCallback 模式。V4 的核心创新包括 Hooks（钩子）和 Flash Accounting（闪电记账）。

```solidity
// SPDX-License-Identifier: MIT  // SPDX 许可证标识符：MIT 许可证
pragma solidity ^0.8.26;  // 版本指令，要求使用 Solidity 0.8.26 及以上版本

// Uniswap V4 PoolManager 主网地址
address constant POOL_MANAGER = 0x000000000004444c5dc75cB358380D2e3dE08A90;
address constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
address constant USDC = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;

// UniswapV4Swap 合约：V4 交换示例
contract UniswapV4Swap is IUnlockCallback {  // 实现 unlock 回调接口
    IPoolManager public immutable poolManager;

    error NotPoolManager();
    error SwapFailed();

    constructor() {
        poolManager = IPoolManager(POOL_MANAGER);
    }

    // swapExactInput 函数：精确输入交换
    function swapExactInput(
        PoolKey calldata key,       // 池键（定义代币对 + 费率 + 钩子）
        uint128 amountIn,           // 输入数量
        uint128 minAmountOut        // 最小输出（滑点保护）
    ) external returns (uint256 amountOut) {
        // 编码参数，通过 unlock 回调传递
        bytes memory data = abi.encode(
            SwapParams({
                key: key, amountIn: amountIn, minAmountOut: minAmountOut,
                zeroForOne: true, sender: msg.sender
            })
        );

        // unlock 触发 → unlockCallback 中执行交换
        bytes memory result = poolManager.unlock(data);
        amountOut = abi.decode(result, (uint256));
    }

    // unlockCallback 函数：PoolManager 回调，真正执行交换逻辑
    function unlockCallback(bytes calldata data)
        external override returns (bytes memory)
    {
        if (msg.sender != address(poolManager)) revert NotPoolManager();

        SwapParams memory params = abi.decode(data, (SwapParams));

        // 执行交换
        // zeroForOne: true = token0 → token1, false = token1 → token0
        // amountSpecified: 负数 = 精确输入, 正数 = 精确输出
        BalanceDelta delta = poolManager.swap(
            params.key,
            IPoolManager.SwapParams({
                zeroForOne: params.zeroForOne,
                amountSpecified: -int256(uint256(params.amountIn)),  // 负数 = 精确输入
                sqrtPriceLimitX96: params.zeroForOne
                    ? MIN_SQRT_PRICE + 1
                    : MAX_SQRT_PRICE - 1
            }),
            bytes("")  // hook 数据（空 = 无钩子回调）
        );

        // 从 delta 中提取输出量
        // delta.amount0() < 0 → 欠池 token0
        // delta.amount1() > 0 → 池欠 token1
        uint256 amountOut = params.zeroForOne
            ? uint256(int256(delta.amount1()))  // token1 输出
            : uint256(int256(delta.amount0())); // token0 输出

        if (amountOut < params.minAmountOut) revert SwapFailed();

        // 结算（Settle）：支付输入的代币
        Currency inputCurrency = params.zeroForOne
            ? params.key.currency0 : params.key.currency1;

        IERC20(Currency.unwrap(inputCurrency)).transferFrom(
            params.sender, address(poolManager), params.amountIn
        );
        poolManager.settle(inputCurrency);  // 结算输入

        // 提取（Take）：接收输出的代币
        Currency outputCurrency = params.zeroForOne
            ? params.key.currency1 : params.key.currency0;

        poolManager.take(outputCurrency, params.sender, amountOut);

        return abi.encode(amountOut);
    }

    struct SwapParams {
        PoolKey key; uint128 amountIn; uint128 minAmountOut;
        bool zeroForOne; address sender;
    }
}

// ===== V4 核心类型 =====

uint160 constant MIN_SQRT_PRICE = 4295128739;
uint160 constant MAX_SQRT_PRICE = 1461446703485210103287273052203988822378723970342;

type Currency is address;  // Currency 类型（address(0) = 原生 ETH）

struct PoolKey {  // 池键：唯一标识一个池
    Currency currency0; Currency currency1;
    uint24 fee; int24 tickSpacing; address hooks;
}

type BalanceDelta is int256;  // 余额变化量

library BalanceDeltaLibrary {
    function amount0(BalanceDelta delta) internal pure returns (int128) {
        return int128(int256(BalanceDelta.unwrap(delta) >> 128));
    }
    function amount1(BalanceDelta delta) internal pure returns (int128) {
        return int128(int256(BalanceDelta.unwrap(delta)));
    }
}

using BalanceDeltaLibrary for BalanceDelta;

interface IPoolManager {
    struct SwapParams {
        bool zeroForOne; int256 amountSpecified; uint160 sqrtPriceLimitX96;
    }
    function unlock(bytes calldata data) external returns (bytes memory);
    function swap(PoolKey memory key, SwapParams memory params, bytes calldata hookData)
        external returns (BalanceDelta);
    function settle(Currency currency) external payable returns (uint256);
    function take(Currency currency, address to, uint256 amount) external;
}

interface IUnlockCallback {
    function unlockCallback(bytes calldata data) external returns (bytes memory);
}

interface IERC20 { /* 标准 ERC20 接口 */ }
```

## V4 核心创新

| 特性 | V3 | V4 |
|------|-----|-----|
| 架构 | 每池独立合约 | 单例 PoolManager |
| 手续费 | 需编码到路径 | PoolKey 中指定 |
| 钩子（Hooks） | 不支持 | 支持自定义回调（swap 前后） |
| 闪电记账 | 不支持 | 支持净额结算，减少转账 |
| 原生 ETH | 需包装为 WETH | 原生支持 ETH |
