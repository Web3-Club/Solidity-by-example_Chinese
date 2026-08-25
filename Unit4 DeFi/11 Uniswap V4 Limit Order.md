# Uniswap V4 限价单

对应英文原页：https://solidity-by-example.org/defi/uniswap-v4-limit-order

Uniswap V4 的 hook 允许在兑换生命周期中执行自定义逻辑。本示例演示使用 `afterSwap` 实现的限价单 hook。

使用 hook 实现限价单的工作方式：

1. 用户调用 `placeLimitOrder()`，指定 tick（价格）和方向
2. 代币由 hook 合约持有
3. 当兑换使价格越过目标 tick 时，会触发 `afterSwap`
4. hook 检测已成交的订单并执行它们
5. 用户收到兑换后的代币

关键 hook 概念：

- **权限（Permissions）** - `getHookPermissions()` 声明你实现了哪些 hook
- **Hook 地址** - 必须根据权限设置特定的位（使用 CREATE2）
- **回调（Callbacks）** - PoolManager 在每个生命周期节点调用你的 hook

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

/// @notice 简化的 Uniswap V4 限价单 Hook
/// @dev Hook 允许在兑换生命周期中执行自定义逻辑
/// 本示例演示使用 afterSwap 实现的基础限价单机制
contract LimitOrderHook is IHooks {
    IPoolManager public immutable poolManager;

    // 映射：poolId => tick => zeroForOne => 总流动性
    mapping(bytes32 => mapping(int24 => mapping(bool => uint256))) public tickLiquidity;

    // 映射：poolId => tick => zeroForOne => 用户 => 流动性
    mapping(bytes32 => mapping(int24 => mapping(bool => mapping(address => uint256))))
        public userPositions;

    error NotPoolManager();
    error InvalidTick();

    constructor(IPoolManager _poolManager) {
        poolManager = _poolManager;
    }

    /// @notice 在指定 tick 下限价单
    /// @param key 要下限价单的资金池
    /// @param tick 限价单的 tick（价格点）
    /// @param zeroForOne true = 卖出 token0 换 token1，false = 卖出 token1 换 token0
    /// @param amount 要卖出的代币数量
    function placeLimitOrder(
        PoolKey calldata key,
        int24 tick,
        bool zeroForOne,
        uint256 amount
    ) external {
        // 校验 tick 位于当前价格的正确一侧
        (, int24 currentTick,,) = poolManager.getSlot0(toId(key));

        // 卖出 token0：tick 必须高于当前价格（价格上涨）
        // 卖出 token1：tick 必须低于当前价格（价格下跌）
        if (zeroForOne && tick <= currentTick) revert InvalidTick();
        if (!zeroForOne && tick >= currentTick) revert InvalidTick();

        bytes32 poolId = toId(key);

        // 从用户转入代币
        Currency currency = zeroForOne ? key.currency0 : key.currency1;
        IERC20(Currency.unwrap(currency)).transferFrom(
            msg.sender,
            address(this),
            amount
        );

        // 记录仓位
        tickLiquidity[poolId][tick][zeroForOne] += amount;
        userPositions[poolId][tick][zeroForOne][msg.sender] += amount;
    }

    /// @notice 每次兑换后由 PoolManager 调用
    /// @dev 检查价格是否越过任何限价单 tick，并执行它们
    function afterSwap(
        address,
        PoolKey calldata key,
        IPoolManager.SwapParams calldata params,
        BalanceDelta,
        bytes calldata
    ) external override returns (bytes4, int128) {
        if (msg.sender != address(poolManager)) revert NotPoolManager();

        (, int24 currentTick,,) = poolManager.getSlot0(toId(key));
        bytes32 poolId = toId(key);

        // 检查该 tick 上是否有应成交的限价单
        // zeroForOne 兑换使价格下降，因此检查卖出 token1 的订单
        // !zeroForOne 兑换使价格上升，因此检查卖出 token0 的订单
        bool checkZeroForOne = !params.zeroForOne;

        uint256 liquidity = tickLiquidity[poolId][currentTick][checkZeroForOne];

        if (liquidity > 0) {
            // 执行该 tick 上的限价单
            _executeLimitOrders(key, currentTick, checkZeroForOne, liquidity);
        }

        return (IHooks.afterSwap.selector, 0);
    }

    /// @notice 执行指定 tick 上的限价单
    function _executeLimitOrders(
        PoolKey calldata key,
        int24 tick,
        bool zeroForOne,
        uint256 amount
    ) internal {
        // 在完整实现中，这里会：
        // 1. 使用 poolManager.swap() 兑换代币
        // 2. 将输出代币分配给下限价单的用户
        // 3. 清除已成交的仓位

        bytes32 poolId = toId(key);

        // 清除该 tick 的流动性（订单已成交）
        tickLiquidity[poolId][tick][zeroForOne] = 0;

        // 发出事件，供链下跟踪
        emit LimitOrderFilled(poolId, tick, zeroForOne, amount);
    }

    /// @notice 用户可以取消未成交的限价单
    function cancelLimitOrder(
        PoolKey calldata key,
        int24 tick,
        bool zeroForOne
    ) external {
        bytes32 poolId = toId(key);
        uint256 amount = userPositions[poolId][tick][zeroForOne][msg.sender];

        require(amount > 0, "No position");

        // 清除仓位
        userPositions[poolId][tick][zeroForOne][msg.sender] = 0;
        tickLiquidity[poolId][tick][zeroForOne] -= amount;

        // 退还代币
        Currency currency = zeroForOne ? key.currency0 : key.currency1;
        IERC20(Currency.unwrap(currency)).transfer(msg.sender, amount);
    }

    /// @notice 返回 hook 权限 - 我们只需要 afterSwap
    function getHookPermissions() public pure returns (Hooks.Permissions memory) {
        return Hooks.Permissions({
            beforeInitialize: false,
            afterInitialize: false,
            beforeAddLiquidity: false,
            afterAddLiquidity: false,
            beforeRemoveLiquidity: false,
            afterRemoveLiquidity: false,
            beforeSwap: false,
            afterSwap: true, // 我们需要这个！
            beforeDonate: false,
            afterDonate: false,
            beforeSwapReturnDelta: false,
            afterSwapReturnDelta: false,
            afterAddLiquidityReturnDelta: false,
            afterRemoveLiquidityReturnDelta: false
        });
    }

    // 计算 pool ID 的辅助函数
    function toId(PoolKey memory key) internal pure returns (bytes32) {
        return keccak256(abi.encode(key));
    }

    // 必需的 hook 接口函数（未使用的 hook 为空实现）
    function beforeInitialize(address, PoolKey calldata, uint160)
        external pure override returns (bytes4) {
        return IHooks.beforeInitialize.selector;
    }
    function afterInitialize(address, PoolKey calldata, uint160, int24)
        external pure override returns (bytes4) {
        return IHooks.afterInitialize.selector;
    }
    function beforeAddLiquidity(address, PoolKey calldata, IPoolManager.ModifyLiquidityParams calldata, bytes calldata)
        external pure override returns (bytes4) {
        return IHooks.beforeAddLiquidity.selector;
    }
    function afterAddLiquidity(address, PoolKey calldata, IPoolManager.ModifyLiquidityParams calldata, BalanceDelta, BalanceDelta, bytes calldata)
        external pure override returns (bytes4, BalanceDelta) {
        return (IHooks.afterAddLiquidity.selector, BalanceDelta.wrap(0));
    }
    function beforeRemoveLiquidity(address, PoolKey calldata, IPoolManager.ModifyLiquidityParams calldata, bytes calldata)
        external pure override returns (bytes4) {
        return IHooks.beforeRemoveLiquidity.selector;
    }
    function afterRemoveLiquidity(address, PoolKey calldata, IPoolManager.ModifyLiquidityParams calldata, BalanceDelta, BalanceDelta, bytes calldata)
        external pure override returns (bytes4, BalanceDelta) {
        return (IHooks.afterRemoveLiquidity.selector, BalanceDelta.wrap(0));
    }
    function beforeSwap(address, PoolKey calldata, IPoolManager.SwapParams calldata, bytes calldata)
        external pure override returns (bytes4, BeforeSwapDelta, uint24) {
        return (IHooks.beforeSwap.selector, BeforeSwapDelta.wrap(0), 0);
    }
    function beforeDonate(address, PoolKey calldata, uint256, uint256, bytes calldata)
        external pure override returns (bytes4) {
        return IHooks.beforeDonate.selector;
    }
    function afterDonate(address, PoolKey calldata, uint256, uint256, bytes calldata)
        external pure override returns (bytes4) {
        return IHooks.afterDonate.selector;
    }

    event LimitOrderFilled(
        bytes32 indexed poolId,
        int24 tick,
        bool zeroForOne,
        uint256 amount
    );
}

// ============ 类型与接口 ============

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

type BalanceDelta is int256;
type BeforeSwapDelta is int256;

library Hooks {
    struct Permissions {
        bool beforeInitialize;
        bool afterInitialize;
        bool beforeAddLiquidity;
        bool afterAddLiquidity;
        bool beforeRemoveLiquidity;
        bool afterRemoveLiquidity;
        bool beforeSwap;
        bool afterSwap;
        bool beforeDonate;
        bool afterDonate;
        bool beforeSwapReturnDelta;
        bool afterSwapReturnDelta;
        bool afterAddLiquidityReturnDelta;
        bool afterRemoveLiquidityReturnDelta;
    }
}

interface IPoolManager {
    struct SwapParams {
        bool zeroForOne;
        int256 amountSpecified;
        uint160 sqrtPriceLimitX96;
    }

    struct ModifyLiquidityParams {
        int24 tickLower;
        int24 tickUpper;
        int256 liquidityDelta;
        bytes32 salt;
    }

    function getSlot0(bytes32 poolId)
        external
        view
        returns (uint160 sqrtPriceX96, int24 tick, uint24 protocolFee, uint24 lpFee);

    function swap(PoolKey memory key, SwapParams memory params, bytes calldata hookData)
        external
        returns (BalanceDelta);
}

interface IHooks {
    function beforeInitialize(address sender, PoolKey calldata key, uint160 sqrtPriceX96)
        external returns (bytes4);
    function afterInitialize(address sender, PoolKey calldata key, uint160 sqrtPriceX96, int24 tick)
        external returns (bytes4);
    function beforeAddLiquidity(address sender, PoolKey calldata key, IPoolManager.ModifyLiquidityParams calldata params, bytes calldata hookData)
        external returns (bytes4);
    function afterAddLiquidity(address sender, PoolKey calldata key, IPoolManager.ModifyLiquidityParams calldata params, BalanceDelta delta, BalanceDelta feesAccrued, bytes calldata hookData)
        external returns (bytes4, BalanceDelta);
    function beforeRemoveLiquidity(address sender, PoolKey calldata key, IPoolManager.ModifyLiquidityParams calldata params, bytes calldata hookData)
        external returns (bytes4);
    function afterRemoveLiquidity(address sender, PoolKey calldata key, IPoolManager.ModifyLiquidityParams calldata params, BalanceDelta delta, BalanceDelta feesAccrued, bytes calldata hookData)
        external returns (bytes4, BalanceDelta);
    function beforeSwap(address sender, PoolKey calldata key, IPoolManager.SwapParams calldata params, bytes calldata hookData)
        external returns (bytes4, BeforeSwapDelta, uint24);
    function afterSwap(address sender, PoolKey calldata key, IPoolManager.SwapParams calldata params, BalanceDelta delta, bytes calldata hookData)
        external returns (bytes4, int128);
    function beforeDonate(address sender, PoolKey calldata key, uint256 amount0, uint256 amount1, bytes calldata hookData)
        external returns (bytes4);
    function afterDonate(address sender, PoolKey calldata key, uint256 amount0, uint256 amount1, bytes calldata hookData)
        external returns (bytes4);
}

interface IERC20 {
    function transferFrom(address sender, address recipient, uint256 amount)
        external returns (bool);
    function transfer(address recipient, uint256 amount)
        external returns (bool);
}

```

---
## 关注我们
[Yanbo的Twitter](https://twitter.com/YanboOfficial)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)

加入Web3Club 官方讨论群：YanboTravelAllWorld（微信号）

[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
