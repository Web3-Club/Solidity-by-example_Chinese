# Uniswap V4 Swap

对应英文原页：https://solidity-by-example.org/defi/uniswap-v4-swap

Uniswap V4 引入了单例（singleton）`PoolManager`，所有资金池都保存在同一个合约中。

与 V3 的主要区别：

- **单例架构（singleton architecture）** - 所有资金池位于同一个合约中
- **闪记（flash accounting）** - 代币转账只在最后发生，从而节省 gas
- **unlock/unlockCallback 模式** - 通过回调进行交互

进行兑换：

1. 调用 `poolManager.unlock()` 并传入编码后的参数
2. PoolManager 调用你的 `unlockCallback()`
3. 在回调中：执行兑换、结算输入、取出输出
4. 在 unlock 完成之前，delta 必须轧差为零

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

// 以太坊主网上的 Uniswap V4 PoolManager
address constant POOL_MANAGER = 0x000000000004444c5dc75cB358380D2e3dE08A90;
address constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
address constant USDC = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;

/// @notice 直接使用 PoolManager 在 Uniswap V4 上兑换的示例
/// @dev V4 使用单例 PoolManager，配合 unlock/unlockCallback 模式
contract UniswapV4Swap is IUnlockCallback {
    IPoolManager public immutable poolManager;

    error NotPoolManager();
    error SwapFailed();

    constructor() {
        poolManager = IPoolManager(POOL_MANAGER);
    }

    /// @notice 按精确输入数量兑换输出代币
    /// @param key 用于标识资金池的 pool key
    /// @param amountIn 要兑换的输入代币数量
    /// @param minAmountOut 可接受的最小输出数量
    function swapExactInput(
        PoolKey calldata key,
        uint128 amountIn,
        uint128 minAmountOut
    ) external returns (uint256 amountOut) {
        // 编码兑换参数，以便通过 unlock 回调传递
        bytes memory data = abi.encode(
            SwapParams({
                key: key,
                amountIn: amountIn,
                minAmountOut: minAmountOut,
                zeroForOne: true,
                sender: msg.sender
            })
        );

        // 启动兑换 - PoolManager 将调用 unlockCallback
        bytes memory result = poolManager.unlock(data);
        amountOut = abi.decode(result, (uint256));
    }

    /// @notice PoolManager 在 unlock 之后的回调
    /// @dev 实际兑换逻辑在这里执行
    function unlockCallback(bytes calldata data)
        external
        override
        returns (bytes memory)
    {
        if (msg.sender != address(poolManager)) revert NotPoolManager();

        SwapParams memory params = abi.decode(data, (SwapParams));

        // 执行兑换
        // zeroForOne: true = token0 -> token1，false = token1 -> token0
        // amountSpecified: 负数 = 精确输入，正数 = 精确输出
        BalanceDelta delta = poolManager.swap(
            params.key,
            IPoolManager.SwapParams({
                zeroForOne: params.zeroForOne,
                amountSpecified: -int256(uint256(params.amountIn)),
                sqrtPriceLimitX96: params.zeroForOne
                    ? MIN_SQRT_PRICE + 1
                    : MAX_SQRT_PRICE - 1
            }),
            bytes("")
        );

        // 从 delta 计算数量
        // delta.amount0() 为负（我们欠池子）
        // delta.amount1() 为正（池子欠我们）
        uint256 amountOut = params.zeroForOne
            ? uint256(int256(delta.amount1()))
            : uint256(int256(delta.amount0()));

        if (amountOut < params.minAmountOut) revert SwapFailed();

        // 结算输入代币（支付我们欠的部分）
        Currency inputCurrency = params.zeroForOne
            ? params.key.currency0
            : params.key.currency1;

        IERC20(Currency.unwrap(inputCurrency)).transferFrom(
            params.sender,
            address(poolManager),
            params.amountIn
        );
        poolManager.settle(inputCurrency);

        // 取出输出代币（领取池子欠我们的部分）
        Currency outputCurrency = params.zeroForOne
            ? params.key.currency1
            : params.key.currency0;

        poolManager.take(outputCurrency, params.sender, amountOut);

        return abi.encode(amountOut);
    }

    struct SwapParams {
        PoolKey key;
        uint128 amountIn;
        uint128 minAmountOut;
        bool zeroForOne;
        address sender;
    }
}

// 兑换的 sqrt 价格限制
uint160 constant MIN_SQRT_PRICE = 4295128739;
uint160 constant MAX_SQRT_PRICE =
    1461446703485210103287273052203988822378723970342;

// Currency 是 address 的包装类型（address(0) = 原生 ETH）
type Currency is address;

library CurrencyLibrary {
    function unwrap(Currency currency) internal pure returns (address) {
        return Currency.unwrap(currency);
    }
}

using CurrencyLibrary for Currency;

struct PoolKey {
    Currency currency0;
    Currency currency1;
    uint24 fee;
    int24 tickSpacing;
    address hooks;
}

/// @notice 兑换操作返回的余额 delta
/// @dev 负数 = 你欠池子，正数 = 池子欠你
type BalanceDelta is int256;

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
        bool zeroForOne;
        int256 amountSpecified;
        uint160 sqrtPriceLimitX96;
    }

    function unlock(bytes calldata data) external returns (bytes memory);
    function swap(PoolKey memory key, SwapParams memory params, bytes calldata hookData)
        external
        returns (BalanceDelta);
    function settle(Currency currency) external payable returns (uint256);
    function take(Currency currency, address to, uint256 amount) external;
}

interface IUnlockCallback {
    function unlockCallback(bytes calldata data) external returns (bytes memory);
}

interface IERC20 {
    function transferFrom(address sender, address recipient, uint256 amount)
        external
        returns (bool);
    function approve(address spender, uint256 amount) external returns (bool);
}

```

---
## 关注我们
[Yanbo的Twitter](https://twitter.com/YanboOfficial)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)

加入Web3Club 官方讨论群：YanboTravelAllWorld（微信号）

[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
