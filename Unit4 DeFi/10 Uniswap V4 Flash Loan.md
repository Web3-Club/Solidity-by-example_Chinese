# Uniswap V4 闪电贷

对应英文原页：https://solidity-by-example.org/defi/uniswap-v4-flash

Uniswap V4 闪电贷是**免费**的——没有手续费！这得益于闪记（flash accounting）。

工作原理：

1. 调用 `poolManager.unlock()` 开始
2. 在回调中调用 `poolManager.take()` 借入代币（产生债务）
3. 将代币用于套利、清算、抵押品兑换等
4. 通过把代币转回并调用 `poolManager.settle()` 来偿还
5. 只要 delta 轧差为零，交易就会成功

V3 会对闪电贷收取手续费；V4 的单例架构和闪记使借款几乎免费——你只需支付 gas。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

// 以太坊主网上的 Uniswap V4 PoolManager
address constant POOL_MANAGER = 0x000000000004444c5dc75cB358380D2e3dE08A90;

/// @notice Uniswap V4 闪电贷示例
/// @dev 由于闪记，V4 闪电贷是免费的——没有手续费！
/// 在 unlock 回调期间借入代币，并在回调结束前偿还
contract UniswapV4Flash is IUnlockCallback {
    IPoolManager public immutable poolManager;

    error NotPoolManager();
    error FlashLoanFailed();

    constructor() {
        poolManager = IPoolManager(POOL_MANAGER);
    }

    /// @notice 执行闪电贷
    /// @param currency 要借入的代币（原生 ETH 使用 address(0)）
    /// @param amount 借入数量
    /// @param data 传给闪电贷逻辑的任意数据
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

        poolManager.unlock(callbackData);
    }

    /// @notice 来自 PoolManager 的回调 - 在此执行闪电贷逻辑
    function unlockCallback(bytes calldata callbackData)
        external
        override
        returns (bytes memory)
    {
        if (msg.sender != address(poolManager)) revert NotPoolManager();

        FlashParams memory params = abi.decode(callbackData, (FlashParams));

        // 从池中取出代币（产生债务）
        poolManager.take(params.currency, address(this), params.amount);

        // ============================================
        // 你的闪电贷逻辑写在这里！
        // 你现在可以使用借入的代币
        // ============================================

        // 示例：调用自定义逻辑
        _executeFlashLoanLogic(params.currency, params.amount, params.data);

        // ============================================
        // 偿还闪电贷
        // ============================================

        // 对于 ERC20：先将代币转入 PoolManager，再 settle
        if (!isNative(params.currency)) {
            IERC20(Currency.unwrap(params.currency)).transfer(
                address(poolManager),
                params.amount
            );
            poolManager.settle(params.currency);
        } else {
            // 对于原生 ETH：带 value 进行 settle
            poolManager.settle{value: params.amount}(params.currency);
        }

        // 没有手续费！delta 现在为零，unlock 将会成功
        return bytes("");
    }

    /// @notice 重写此函数以实现你的闪电贷逻辑
    function _executeFlashLoanLogic(
        Currency currency,
        uint256 amount,
        bytes memory data
    ) internal virtual {
        // 示例：套利、清算、抵押品兑换等
        // 借入的代币位于本合约中
    }

    function isNative(Currency currency) internal pure returns (bool) {
        return Currency.unwrap(currency) == address(0);
    }

    // 允许接收 ETH
    receive() external payable {}

    struct FlashParams {
        Currency currency;
        uint256 amount;
        address sender;
        bytes data;
    }
}

// Currency 是 address 的包装类型（address(0) = 原生 ETH）
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

interface IERC20 {
    function transfer(address recipient, uint256 amount)
        external
        returns (bool);
    function balanceOf(address account) external view returns (uint256);
}

```

---
## 关注我们
[Yanbo的Twitter](https://x.com/Yanbo2004)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)


[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
