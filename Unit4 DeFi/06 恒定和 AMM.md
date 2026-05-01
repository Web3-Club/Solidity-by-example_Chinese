# 恒定和 AMM（CSAMM）

恒定和自动做市商（Constant Sum AMM）遵循 `x + y = k` 的不变量。相比恒定乘积 AMM，价格始终恒定（不存在滑点），但流动性可能在极端价格时耗尽。

```solidity
// SPDX-License-Identifier: MIT  // SPDX 许可证标识符：MIT 许可证
pragma solidity ^0.8.26;  // 版本指令，要求使用 Solidity 0.8.26 及以上版本

// CSAMM 合约：恒定和自动做市商（x + y = k）
contract CSAMM {  // 定义 CSAMM 合约
    IERC20 public immutable token0;  // 代币 0
    IERC20 public immutable token1;  // 代币 1

    uint256 public reserve0;  // 代币 0 储备
    uint256 public reserve1;  // 代币 1 储备

    uint256 public totalSupply;                   // LP 代币总供应
    mapping(address => uint256) public balanceOf;  // LP 代币余额

    constructor(address _token0, address _token1) {
        // 注意：此合约假设 token0 和 token1 的小数位相同
        token0 = IERC20(_token0);
        token1 = IERC20(_token1);
    }

    function _mint(address _to, uint256 _amount) private {
        balanceOf[_to] += _amount;
        totalSupply += _amount;
    }

    function _burn(address _from, uint256 _amount) private {
        balanceOf[_from] -= _amount;
        totalSupply -= _amount;
    }

    function _update(uint256 _res0, uint256 _res1) private {
        reserve0 = _res0;
        reserve1 = _res1;
    }

    // swap 函数：恒定和交换（1:1 价格，仅扣手续费）
    function swap(address _tokenIn, uint256 _amountIn)
        external
        returns (uint256 amountOut)
    {
        require(
            _tokenIn == address(token0) || _tokenIn == address(token1),
            "invalid token"
        );

        bool isToken0 = _tokenIn == address(token0);
        (IERC20 tokenIn, IERC20 tokenOut, uint256 resIn, uint256 resOut) =
        isToken0
            ? (token0, token1, reserve0, reserve1)
            : (token1, token0, reserve1, reserve0);

        tokenIn.transferFrom(msg.sender, address(this), _amountIn);
        uint256 amountIn = tokenIn.balanceOf(address(this)) - resIn;

        // 0.3% 手续费
        amountOut = (amountIn * 997) / 1000;  // 扣除手续费后的输出

        (uint256 res0, uint256 res1) = isToken0
            ? (resIn + amountIn, resOut - amountOut)
            : (resOut - amountOut, resIn + amountIn);

        _update(res0, res1);
        tokenOut.transfer(msg.sender, amountOut);
    }

    // addLiquidity 函数：添加两种代币的流动性
    function addLiquidity(uint256 _amount0, uint256 _amount1)
        external
        returns (uint256 shares)
    {
        token0.transferFrom(msg.sender, address(this), _amount0);
        token1.transferFrom(msg.sender, address(this), _amount1);

        uint256 bal0 = token0.balanceOf(address(this));
        uint256 bal1 = token1.balanceOf(address(this));

        uint256 d0 = bal0 - reserve0;
        uint256 d1 = bal1 - reserve1;

        // s = (d0 + d1) * T / (reserve0 + reserve1)
        if (totalSupply > 0) {
            shares = ((d0 + d1) * totalSupply) / (reserve0 + reserve1);
        } else {
            shares = d0 + d1;  // 首次添加，shares = 总增加值
        }

        require(shares > 0, "shares = 0");
        _mint(msg.sender, shares);
        _update(bal0, bal1);
    }

    // removeLiquidity 函数：移除流动性
    function removeLiquidity(uint256 _shares)
        external
        returns (uint256 d0, uint256 d1)
    {
        // a = L * s / T = (reserve0 + reserve1) * s / T
        // 按比例提取两种代币
        d0 = (reserve0 * _shares) / totalSupply;
        d1 = (reserve1 * _shares) / totalSupply;

        _burn(msg.sender, _shares);
        _update(reserve0 - d0, reserve1 - d1);

        if (d0 > 0) token0.transfer(msg.sender, d0);
        if (d1 > 0) token1.transfer(msg.sender, d1);
    }
}

interface IERC20 { /* 标准 ERC20 接口 */ }
```

## 与 CPAMM 对比

| 特性 | 恒定和 (CSAMM) | 恒定乘积 (CPAMM) |
|------|-------------|---------------|
| 不变量 | `x + y = k` | `x * y = k` |
| 价格曲线 | 线性（无滑点） | 双曲线（有滑点） |
| 流动性枯竭 | 可能完全耗尽某一种代币 | 永不完全耗尽 |
| 适用场景 | 稳定币对（价格接近 1:1） | 波动资产对 |
