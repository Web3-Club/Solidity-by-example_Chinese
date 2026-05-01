# Uniswap V4 限价单钩子

Uniswap V4 的 Hooks（钩子）机制允许在交换生命周期的关键节点执行自定义逻辑。本示例实现一个限价单钩子：当价格到达指定 tick 时自动执行挂单。

```solidity
// SPDX-License-Identifier: MIT  // SPDX 许可证标识符：MIT 许可证
pragma solidity ^0.8.26;  // 版本指令，要求使用 Solidity 0.8.26 及以上版本

// LimitOrderHook 合约：V4 限价单钩子
contract LimitOrderHook is IHooks {  // 实现 IHooks 接口
    IPoolManager public immutable poolManager;

    // 池 → tick → zeroForOne → 总流动性
    mapping(bytes32 => mapping(int24 => mapping(bool => uint256))) public tickLiquidity;
    // 池 → tick → zeroForOne → 用户 → 流动性
    mapping(bytes32 => mapping(int24 => mapping(bool => mapping(address => uint256))))
        public userPositions;

    error NotPoolManager();
    error InvalidTick();

    constructor(IPoolManager _poolManager) {
        poolManager = _poolManager;
    }

    // placeLimitOrder 函数：在指定 tick 处挂限价单
    function placeLimitOrder(
        PoolKey calldata key,      // 池键值
        int24 tick,                // 目标价格 tick
        bool zeroForOne,           // true = 卖 token0 买 token1
        uint256 amount             // 卖出的代币数量
    ) external {
        // 验证 tick 在当前价格的正确一侧
        (, int24 currentTick,,) = poolManager.getSlot0(toId(key));

        // 卖 token0 → tick 必须高于当前（价格上升时成交）
        // 卖 token1 → tick 必须低于当前（价格下降时成交）
        if (zeroForOne && tick <= currentTick) revert InvalidTick();
        if (!zeroForOne && tick >= currentTick) revert InvalidTick();

        bytes32 poolId = toId(key);

        // 从用户转入代币
        Currency currency = zeroForOne ? key.currency0 : key.currency1;
        IERC20(Currency.unwrap(currency)).transferFrom(msg.sender, address(this), amount);

        // 记录仓位
        tickLiquidity[poolId][tick][zeroForOne] += amount;
        userPositions[poolId][tick][zeroForOne][msg.sender] += amount;
    }

    // afterSwap 钩子：每次交换后检查是否到达限价单价格
    function afterSwap(
        address,                   // sender
        PoolKey calldata key,      // 池键值
        IPoolManager.SwapParams calldata params,  // 交换参数
        BalanceDelta,              // 交换的余额变化
        bytes calldata             // hook 数据
    ) external override returns (bytes4, int128) {
        if (msg.sender != address(poolManager)) revert NotPoolManager();

        (, int24 currentTick,,) = poolManager.getSlot0(toId(key));
        bytes32 poolId = toId(key);

        // zeroForOne swap 使价格下降 → 检查卖 token1 的挂单
        // !zeroForOne swap 使价格上涨 → 检查卖 token0 的挂单
        bool checkZeroForOne = !params.zeroForOne;

        uint256 liquidity = tickLiquidity[poolId][currentTick][checkZeroForOne];

        if (liquidity > 0) {
            _executeLimitOrders(key, currentTick, checkZeroForOne, liquidity);
        }

        return (IHooks.afterSwap.selector, 0);
    }

    function _executeLimitOrders(
        PoolKey calldata key, int24 tick, bool zeroForOne, uint256 amount
    ) internal {
        bytes32 poolId = toId(key);
        tickLiquidity[poolId][tick][zeroForOne] = 0;  // 清除挂单
        emit LimitOrderFilled(poolId, tick, zeroForOne, amount);
    }

    // cancelLimitOrder 函数：取消未成交的限价单
    function cancelLimitOrder(
        PoolKey calldata key, int24 tick, bool zeroForOne
    ) external {
        bytes32 poolId = toId(key);
        uint256 amount = userPositions[poolId][tick][zeroForOne][msg.sender];
        require(amount > 0, "No position");

        userPositions[poolId][tick][zeroForOne][msg.sender] = 0;
        tickLiquidity[poolId][tick][zeroForOne] -= amount;

        Currency currency = zeroForOne ? key.currency0 : key.currency1;
        IERC20(Currency.unwrap(currency)).transfer(msg.sender, amount);
    }

    // getHookPermissions 函数：声明需要的钩子权限
    function getHookPermissions() public pure returns (Hooks.Permissions memory) {
        return Hooks.Permissions({
            beforeInitialize: false, afterInitialize: false,
            beforeAddLiquidity: false, afterAddLiquidity: false,
            beforeRemoveLiquidity: false, afterRemoveLiquidity: false,
            beforeSwap: false, afterSwap: true,  // 仅需 afterSwap
            beforeDonate: false, afterDonate: false,
            beforeSwapReturnDelta: false, afterSwapReturnDelta: false,
            afterAddLiquidityReturnDelta: false, afterRemoveLiquidityReturnDelta: false
        });
    }

    function toId(PoolKey memory key) internal pure returns (bytes32) {
        return keccak256(abi.encode(key));
    }

    // 未使用的钩子（均返回 selector）
    function beforeInitialize(address, PoolKey calldata, uint160) external pure override returns (bytes4) {
        return IHooks.beforeInitialize.selector;
    }
    // ... (其余未使用钩子同理，返回各自 selector)

    event LimitOrderFilled(bytes32 indexed poolId, int24 tick, bool zeroForOne, uint256 amount);
}

// ===== V4 核心类型和接口 =====

type Currency is address;
type BalanceDelta is int256;

struct PoolKey {
    Currency currency0; Currency currency1;
    uint24 fee; int24 tickSpacing; address hooks;
}

library Hooks {
    struct Permissions {
        bool beforeInitialize; bool afterInitialize;
        bool beforeAddLiquidity; bool afterAddLiquidity;
        bool beforeRemoveLiquidity; bool afterRemoveLiquidity;
        bool beforeSwap; bool afterSwap;
        bool beforeDonate; bool afterDonate;
        bool beforeSwapReturnDelta; bool afterSwapReturnDelta;
        bool afterAddLiquidityReturnDelta; bool afterRemoveLiquidityReturnDelta;
    }
}

interface IPoolManager {
    struct SwapParams {
        bool zeroForOne; int256 amountSpecified; uint160 sqrtPriceLimitX96;
    }
    function getSlot0(bytes32 poolId) external view
        returns (uint160 sqrtPriceX96, int24 tick, uint24 protocolFee, uint24 lpFee);
}

interface IHooks {
    function beforeSwap(address, PoolKey calldata, IPoolManager.SwapParams calldata, bytes calldata)
        external returns (bytes4, BeforeSwapDelta, uint24);
    function afterSwap(address, PoolKey calldata, IPoolManager.SwapParams calldata, BalanceDelta, bytes calldata)
        external returns (bytes4, int128);
    // ... 其余 12 个钩子函数
}

interface IERC20 { /* 标准 ERC20 接口 */ }
```

## V4 Hooks 机制

| 钩子 | 触发时机 | 典型用途 |
|------|----------|----------|
| `beforeSwap` | 交换即将执行 | 自定义费率、价格验证 |
| `afterSwap` | 交换刚执行完 | 限价单、TWAP 更新 |
| `beforeAddLiquidity` | 添加流动性前 | 白名单检查 |
| `afterAddLiquidity` | 添加流动性后 | 仓位跟踪 |
| `beforeDonate` | 捐赠前 | 费率重分配 |

**Hooks 设计原则**：一个合约可以实现任意钩子组合，通过 `getHookPermissions` 声明所需权限，无效的钩子不可注册。
