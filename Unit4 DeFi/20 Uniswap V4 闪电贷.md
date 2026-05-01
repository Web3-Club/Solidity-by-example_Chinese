# Uniswap V4 闪电贷

Uniswap V4 的闪电贷**完全免费**（无手续费）。V4 采用闪电记账（Flash Accounting）机制，只需在 unlock 回调期间保持净余额为零即可。

```solidity
// SPDX-License-Identifier: MIT  // SPDX 许可证标识符：MIT 许可证
pragma solidity ^0.8.26;  // 版本指令，要求使用 Solidity 0.8.26 及以上版本

// Uniswap V4 PoolManager 主网地址
address constant POOL_MANAGER = 0x000000000004444c5dc75cB358380D2e3dE08A90;

// UniswapV4Flash 合约：V4 免费闪电贷
// V4 闪电贷因闪电记账机制完全免费 — 无需手续费！
contract UniswapV4Flash is IUnlockCallback {  // 实现 unlock 回调接口
    IPoolManager public immutable poolManager;

    error NotPoolManager();
    error FlashLoanFailed();

    constructor() {
        poolManager = IPoolManager(POOL_MANAGER);
    }

    // flash 函数：发起闪电贷
    // currency: 借入的代币（address(0) = 原生 ETH）
    function flash(Currency currency, uint256 amount, bytes calldata data)
        external
    {
        bytes memory callbackData = abi.encode(
            FlashParams({
                currency: currency,
                amount: amount,
                sender: msg.sender,
                data: data
            })
        );

        poolManager.unlock(callbackData);  // 触发 unlock → unlockCallback
    }

    // unlockCallback 函数：PoolManager 回调
    function unlockCallback(bytes calldata callbackData)
        external override returns (bytes memory)
    {
        if (msg.sender != address(poolManager)) revert NotPoolManager();

        FlashParams memory params = abi.decode(callbackData, (FlashParams));

        // 从池中"取走"代币（产生负余额/债务）
        poolManager.take(params.currency, address(this), params.amount);

        // ============================================
        // 在此执行闪电贷自定义逻辑
        // ============================================

        // 归还闪电贷
        if (!isNative(params.currency)) {
            // ERC20：先转账给 PoolManager，再调用 settle
            IERC20(Currency.unwrap(params.currency)).transfer(
                address(poolManager), params.amount
            );
            poolManager.settle(params.currency);
        } else {
            // 原生 ETH：直接附带 value 调用 settle
            poolManager.settle{value: params.amount}(params.currency);
        }

        // 无手续费！余额差值为零，unlock 成功
        return bytes("");
    }

    function isNative(Currency currency) internal pure returns (bool) {
        return Currency.unwrap(currency) == address(0);
    }

    receive() external payable {}

    struct FlashParams {
        Currency currency; uint256 amount; address sender; bytes data;
    }
}

// ===== V4 核心类型 =====

type Currency is address;

library CurrencyLibrary {
    function unwrap(Currency currency) internal pure returns (address) {
        return Currency.unwrap(currency);
    }
}

using CurrencyLibrary for Currency;

interface IPoolManager {
    function unlock(bytes calldata data) external returns (bytes memory);
    function settle(Currency currency) external payable returns (uint256);
    function take(Currency currency, address to, uint256 amount) external;
}

interface IUnlockCallback {
    function unlockCallback(bytes calldata data) external returns (bytes memory);
}

interface IERC20 { /* 标准 ERC20 接口 */ }
```

## V4 闪电贷 vs V3 闪电贷

| 特性 | V3 Flash | V4 Flash |
|------|----------|----------|
| 手续费 | 按池费率收取（如 0.3%）| **完全免费** |
| 机制 | `pool.flash(recipient, amount0, amount1, data)` | `poolManager.unlock(data)` |
| 借入方式 | 通过回调参数直接获得代币 | `poolManager.take()` 提取 |
| ETH 支持 | 需 WETH | 原生支持 |
| 闪电记账 | 不支持 | 支持净额结算 |
