# 恒定乘积 AMM（CPAMM）

恒定乘积自动做市商（Constant Product AMM）是 Uniswap V2 的核心机制。遵循 `x * y = k` 的不变量，支持 swap、添加流动性和移除流动性。

```solidity
// SPDX-License-Identifier: MIT  // SPDX 许可证标识符：MIT 许可证
pragma solidity ^0.8.26;  // 版本指令，要求使用 Solidity 0.8.26 及以上版本

// CPAMM 合约：恒定乘积自动做市商
contract CPAMM {  // 定义 CPAMM 合约
    IERC20 public immutable token0;  // 代币 0（不可变）
    IERC20 public immutable token1;  // 代币 1（不可变）

    uint256 public reserve0;  // 代币 0 的储备量
    uint256 public reserve1;  // 代币 1 的储备量

    uint256 public totalSupply;                   // LP 代币总供应量
    mapping(address => uint256) public balanceOf;  // LP 代币余额

    constructor(address _token0, address _token1) {
        token0 = IERC20(_token0);
        token1 = IERC20(_token1);
    }

    // _mint 内部函数：铸造 LP 代币
    function _mint(address _to, uint256 _amount) private {
        balanceOf[_to] += _amount;  // 增加用户余额
        totalSupply += _amount;     // 增加总供应量
    }

    // _burn 内部函数：销毁 LP 代币
    function _burn(address _from, uint256 _amount) private {
        balanceOf[_from] -= _amount;  // 减少用户余额
        totalSupply -= _amount;       // 减少总供应量
    }

    // _update 内部函数：更新储备量
    function _update(uint256 _reserve0, uint256 _reserve1) private {
        reserve0 = _reserve0;
        reserve1 = _reserve1;
    }

    // swap 函数：用代币交换另一种代币
    function swap(address _tokenIn, uint256 _amountIn)
        external
        returns (uint256 amountOut)  // 返回换出的数量
    {
        require(
            _tokenIn == address(token0) || _tokenIn == address(token1),
            "invalid token"
        );
        require(_amountIn > 0, "amount in = 0");

        bool isToken0 = _tokenIn == address(token0);  // 判断输入是哪个代币
        (IERC20 tokenIn, IERC20 tokenOut, uint256 reserveIn, uint256 reserveOut)
        = isToken0
            ? (token0, token1, reserve0, reserve1)
            : (token1, token0, reserve1, reserve0);

        tokenIn.transferFrom(msg.sender, address(this), _amountIn);

        // 恒定乘积公式推导：
        // xy = k
        // (x + dx)(y - dy) = k
        // y - dy = k / (x + dx)
        // dy = ydx / (x + dx)
        // 0.3% 手续费
        uint256 amountInWithFee = (_amountIn * 997) / 1000;  // 扣除 0.3% 手续费
        amountOut =
            (reserveOut * amountInWithFee) / (reserveIn + amountInWithFee);

        tokenOut.transfer(msg.sender, amountOut);  // 将输出代币发给用户

        _update(
            token0.balanceOf(address(this)), token1.balanceOf(address(this))
        );
    }

    // addLiquidity 函数：添加流动性
    function addLiquidity(uint256 _amount0, uint256 _amount1)
        external
        returns (uint256 shares)  // 返回铸造的 LP 代币数量
    {
        token0.transferFrom(msg.sender, address(this), _amount0);
        token1.transferFrom(msg.sender, address(this), _amount1);

        // 添加 dx, dy 时需保持价格不变：x/y = (x+dx)/(y+dy) → dy = y/x * dx
        if (reserve0 > 0 || reserve1 > 0) {
            require(
                reserve0 * _amount1 == reserve1 * _amount0, "x / y != dx / dy"
            );
        }

        // LP 代币铸造量推导：
        // f(x, y) = sqrt(xy)（流动性价值函数）
        // L0 = f(x, y), L1 = f(x+dx, y+dy)
        // s = (L1 - L0) / L0 * T = dx/x * T = dy/y * T
        if (totalSupply == 0) {
            shares = _sqrt(_amount0 * _amount1);  // 首次添加，shares = sqrt(dx*dy)
        } else {
            shares = _min(
                (_amount0 * totalSupply) / reserve0,  // s = dx/x * T
                (_amount1 * totalSupply) / reserve1   // s = dy/y * T（取最小值防操纵）
            );
        }
        require(shares > 0, "shares = 0");
        _mint(msg.sender, shares);

        _update(
            token0.balanceOf(address(this)), token1.balanceOf(address(this))
        );
    }

    // removeLiquidity 函数：移除流动性（按比例提取两种代币）
    function removeLiquidity(uint256 _shares)
        external
        returns (uint256 amount0, uint256 amount1)
    {
        // 移除的流动性量：dx = s/T * x, dy = s/T * y
        uint256 bal0 = token0.balanceOf(address(this));
        uint256 bal1 = token1.balanceOf(address(this));

        amount0 = (_shares * bal0) / totalSupply;  // 应提取的 token0
        amount1 = (_shares * bal1) / totalSupply;  // 应提取的 token1
        require(amount0 > 0 && amount1 > 0, "amount0 or amount1 = 0");

        _burn(msg.sender, _shares);
        _update(bal0 - amount0, bal1 - amount1);

        token0.transfer(msg.sender, amount0);
        token1.transfer(msg.sender, amount1);
    }

    // _sqrt 内部函数：牛顿迭代法计算平方根
    function _sqrt(uint256 y) private pure returns (uint256 z) {
        if (y > 3) {
            z = y;
            uint256 x = y / 2 + 1;
            while (x < z) {
                z = x;
                x = (y / x + x) / 2;  // 牛顿迭代
            }
        } else if (y != 0) {
            z = 1;
        }
    }

    function _min(uint256 x, uint256 y) private pure returns (uint256) {
        return x <= y ? x : y;
    }
}

interface IERC20 { /* 标准 ERC20 接口 */ }
```

## 核心公式

| 操作 | 公式 |
|------|------|
| **Swap 输出量** | `dy = y * dx / (x + dx)` |
| **Swap 手续费** | 0.3%（`amountInWithFee = dx * 997 / 1000`）|
| **添加流动性** | `s = min(dx/x * T, dy/y * T)` |
| **移除流动性** | `dx = s/T * x, dy = s/T * y` |
