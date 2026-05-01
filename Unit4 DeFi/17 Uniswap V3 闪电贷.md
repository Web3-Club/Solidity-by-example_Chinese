# Uniswap V3 闪电贷

Uniswap V3 支持原生的闪电贷（Flash）功能，允许用户在单笔交易中借出任意数量的代币，使用后归还本金和手续费。

```solidity
// SPDX-License-Identifier: MIT  // SPDX 许可证标识符：MIT 许可证
pragma solidity ^0.8.26;  // 版本指令，要求使用 Solidity 0.8.26 及以上版本

// UniswapV3Flash 合约：Uniswap V3 闪电贷示例
contract UniswapV3Flash {  // 定义 UniswapV3Flash 合约
    struct FlashCallbackData {  // 回调数据结构
        uint256 amount0;  // 借入的 token0 数量
        uint256 amount1;  // 借入的 token1 数量
        address caller;   // 发起调用者（用于归还手续费）
    }

    IUniswapV3Pool private immutable pool;  // Uniswap V3 池
    IERC20 private immutable token0;        // 代币 0
    IERC20 private immutable token1;        // 代币 1

    constructor(address _pool) {
        pool = IUniswapV3Pool(_pool);
        token0 = IERC20(pool.token0());  // 从池中读取代币地址
        token1 = IERC20(pool.token1());
    }

    // flash 函数：发起闪电贷
    function flash(uint256 amount0, uint256 amount1) external {
        bytes memory data = abi.encode(
            FlashCallbackData({
                amount0: amount0,
                amount1: amount1,
                caller: msg.sender
            })
        );
        // 调用池合约的 flash 函数
        IUniswapV3Pool(pool).flash(address(this), amount0, amount1, data);
    }

    // uniswapV3FlashCallback 函数：闪电贷回调
    function uniswapV3FlashCallback(
        uint256 fee0,  // token0 的手续费（池费率 × 借入量）
        uint256 fee1,  // token1 的手续费
        bytes calldata data
    ) external {
        require(msg.sender == address(pool), "not authorized");

        FlashCallbackData memory decoded = abi.decode(data, (FlashCallbackData));

        // 在此编写自定义逻辑（套利、清算等）

        // 从调用者转入手续费
        if (fee0 > 0) {
            token0.transferFrom(decoded.caller, address(this), fee0);
        }
        if (fee1 > 0) {
            token1.transferFrom(decoded.caller, address(this), fee1);
        }

        // 归还借入代币 + 手续费
        if (fee0 > 0) {
            token0.transfer(address(pool), decoded.amount0 + fee0);
        }
        if (fee1 > 0) {
            token1.transfer(address(pool), decoded.amount1 + fee1);
        }
    }
}

interface IUniswapV3Pool {
    function token0() external view returns (address);
    function token1() external view returns (address);
    function flash(address recipient, uint256 amount0, uint256 amount1, bytes calldata data) external;
}
interface IERC20 { /* 标准 ERC20 接口 */ }
```

## V3 闪电贷特点

- **多种费率等级**：不同的池有不同费率（0.05%、0.30%、1.00%）
- **可同时借入两种代币**：`flash(amount0, amount1)` 可借入任意组合
- **集中流动性**：V3 的流动性集中机制使得大额闪电贷的深度更好
