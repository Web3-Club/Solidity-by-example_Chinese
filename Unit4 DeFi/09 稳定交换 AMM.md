# 稳定交换 AMM（StableSwap）

稳定交换 AMM（StableSwap）是 Curve Finance 的核心算法，结合了恒定和（CSAMM）与恒定乘积（CPAMM）的优点，在价格接近 1:1 时滑点极低。通过牛顿迭代法求解不变量方程。

```solidity
// SPDX-License-Identifier: MIT  // SPDX 许可证标识符：MIT 许可证
pragma solidity ^0.8.26;  // 版本指令，要求使用 Solidity 0.8.26 及以上版本

/*
不变量方程：An^n ∑x_i + D = ADn^n + D^(n+1) / (n^n ∏x_i)

核心知识点：
0. 牛顿迭代法: x_(n+1) = x_n - f(x_n) / f'(x_n)
1. 不变量 D 的计算
2. 交换（Swap）- 计算 Y 和 D
3. 虚拟价格
4. 添加流动性 - 不平衡费率
5. 移除流动性
6. 单币移除流动性 - getYD
*/

library Math {
    function abs(uint256 x, uint256 y) internal pure returns (uint256) {
        return x >= y ? x - y : y - x;
    }
}

contract StableSwap {  // 定义 StableSwap 合约
    uint256 private constant N = 3;              // 代币数量
    // 放大系数 × N^(N-1)，值越大曲线越平
    uint256 private constant A = 1000 * (N ** (N - 1));
    uint256 private constant SWAP_FEE = 300;     // 0.03% 交换费率
    // 流动性费率由 2 个约束推导：
    // 1. 在平衡池中添加/移除流动性时费率为 0
    // 2. 在平衡池中交换 ≈ 先添加再移除流动性
    uint256 private constant LIQUIDITY_FEE = (SWAP_FEE * N) / (4 * (N - 1));
    uint256 private constant FEE_DENOMINATOR = 1e6;  // 费率分母

    address[N] public tokens;      // 代币地址列表
    // 精度乘数（将不同小数的代币统一到 18 位）
    uint256[N] private multipliers = [1, 1e12, 1e12];  // DAI(18)、USDC(6)、USDT(6)
    uint256[N] public balances;    // 各代币余额

    uint256 private constant DECIMALS = 18;
    uint256 public totalSupply;                 // LP 代币总供应
    mapping(address => uint256) public balanceOf;  // LP 代币余额

    constructor(address[N] memory _tokens) {
        tokens = _tokens;
    }

    function _mint(address _to, uint256 _amount) private {
        balanceOf[_to] += _amount;
        totalSupply += _amount;
    }

    function _burn(address _from, uint256 _amount) private {
        balanceOf[_from] -= _amount;
        totalSupply -= _amount;
    }

    // _xp 函数：将余额调整为精度统一的 18 位值
    function _xp() private view returns (uint256[N] memory xp) {
        for (uint256 i; i < N; ++i) {
            xp[i] = balances[i] * multipliers[i];  // 乘以精度乘数
        }
    }

    // _getD 函数：牛顿迭代法计算 D（平衡池中各代币之和）
    function _getD(uint256[N] memory xp) private pure returns (uint256) {
        uint256 a = A * N;  // An^n

        uint256 s;  // s = ∑x_i
        for (uint256 i; i < N; ++i) {
            s += xp[i];
        }

        // 牛顿迭代法，初始猜测 d ≤ s
        uint256 d = s;
        uint256 d_prev;
        for (uint256 i; i < 255; ++i) {
            // p = D^(n+1) / (n^n * ∏x_i)
            uint256 p = d;
            for (uint256 j; j < N; ++j) {
                p = (p * d) / (N * xp[j]);
            }
            d_prev = d;
            // D_(n+1) = (ADn^n + np) * D_n / ((A-1)D_n + (n+1)p)
            d = ((a * s + N * p) * d) / ((a - 1) * d + (N + 1) * p);

            if (Math.abs(d, d_prev) <= 1) {
                return d;  // 收敛
            }
        }
        revert("D didn't converge");
    }

    // _getY 函数：已知代币 i 的新余额 x，计算代币 j 的新余额 y
    function _getY(uint256 i, uint256 j, uint256 x, uint256[N] memory xp)
        private pure returns (uint256)
    {
        uint256 a = A * N;
        uint256 d = _getD(xp);
        uint256 s;
        uint256 c = d;

        uint256 _x;
        for (uint256 k; k < N; ++k) {
            if (k == i) { _x = x; }
            else if (k == j) { continue; }
            else { _x = xp[k]; }

            s += _x;
            c = (c * d) / (N * _x);
        }
        c = (c * d) / (N * a);
        uint256 b = s + d / a;

        // 牛顿迭代法求 y，初始猜测 y ≤ d
        uint256 y_prev;
        uint256 y = d;
        for (uint256 _i; _i < 255; ++_i) {
            y_prev = y;
            // y_(n+1) = (y_n² + c) / (2y_n + b - D)
            y = (y * y + c) / (2 * y + b - d);
            if (Math.abs(y, y_prev) <= 1) {
                return y;
            }
        }
        revert("y didn't converge");
    }

    // getVirtualPrice 函数：估算每份额代币的价值
    function getVirtualPrice() external view returns (uint256) {
        uint256 d = _getD(_xp());
        uint256 _totalSupply = totalSupply;
        if (_totalSupply > 0) {
            return (d * 10 ** DECIMALS) / _totalSupply;
        }
        return 0;
    }

    // swap 函数：交换 dx 数量的 token i 为 token j
    function swap(uint256 i, uint256 j, uint256 dx, uint256 minDy)
        external returns (uint256 dy)
    {
        require(i != j, "i = j");

        IERC20(tokens[i]).transferFrom(msg.sender, address(this), dx);

        uint256[N] memory xp = _xp();
        uint256 x = xp[i] + dx * multipliers[i];

        uint256 y0 = xp[j];
        uint256 y1 = _getY(i, j, x, xp);
        // y0 >= y1（因为 x 增加了），-1 向下取整
        dy = (y0 - y1 - 1) / multipliers[j];

        // 扣除交换费
        uint256 fee = (dy * SWAP_FEE) / FEE_DENOMINATOR;
        dy -= fee;
        require(dy >= minDy, "dy < min");

        balances[i] += dx;
        balances[j] -= dy;

        IERC20(tokens[j]).transfer(msg.sender, dy);
    }

    function addLiquidity(uint256[N] calldata amounts, uint256 minShares)
        external returns (uint256 shares)
    {
        uint256 _totalSupply = totalSupply;
        uint256 d0;
        uint256[N] memory old_xs = _xp();
        if (_totalSupply > 0) {
            d0 = _getD(old_xs);  // 当前流动性 D
        }

        // 转入代币并计算新的精度调整余额
        uint256[N] memory new_xs;
        for (uint256 i; i < N; ++i) {
            uint256 amount = amounts[i];
            if (amount > 0) {
                IERC20(tokens[i]).transferFrom(msg.sender, address(this), amount);
                new_xs[i] = old_xs[i] + amount * multipliers[i];
            } else {
                new_xs[i] = old_xs[i];
            }
        }

        uint256 d1 = _getD(new_xs);  // 添加后的 D
        require(d1 > d0, "liquidity didn't increase");

        // 重新计算 D 以包含不平衡费
        uint256 d2;
        if (_totalSupply > 0) {
            for (uint256 i; i < N; ++i) {
                uint256 idealBalance = (old_xs[i] * d1) / d0;  // 理想比例
                uint256 diff = Math.abs(new_xs[i], idealBalance);
                new_xs[i] -= (LIQUIDITY_FEE * diff) / FEE_DENOMINATOR;  // 扣费
            }
            d2 = _getD(new_xs);  // 扣费后的 D
        } else {
            d2 = d1;  // 首次添加无费率
        }

        for (uint256 i; i < N; ++i) {
            balances[i] += amounts[i];  // 更新余额
        }

        // shares = (d2 - d0) / d0 * totalSupply
        if (_totalSupply > 0) {
            shares = ((d2 - d0) * _totalSupply) / d0;
        } else {
            shares = d2;  // 首次添加
        }
        require(shares >= minShares, "shares < min");
        _mint(msg.sender, shares);
    }

    function removeLiquidity(uint256 shares, uint256[N] calldata minAmountsOut)
        external returns (uint256[N] memory amountsOut)
    {
        uint256 _totalSupply = totalSupply;

        for (uint256 i; i < N; ++i) {
            uint256 amountOut = (balances[i] * shares) / _totalSupply;
            require(amountOut >= minAmountsOut[i], "out < min");
            balances[i] -= amountOut;
            amountsOut[i] = amountOut;
            IERC20(tokens[i]).transfer(msg.sender, amountOut);
        }

        _burn(msg.sender, shares);
    }

    // removeLiquidityOneToken 函数：单币移除流动性
    function removeLiquidityOneToken(uint256 shares, uint256 i, uint256 minAmountOut)
        external returns (uint256 amountOut)
    {
        (amountOut,) = _calcWithdrawOneToken(shares, i);
        require(amountOut >= minAmountOut, "out < min");

        balances[i] -= amountOut;
        _burn(msg.sender, shares);
        IERC20(tokens[i]).transfer(msg.sender, amountOut);
    }

    function _calcWithdrawOneToken(uint256 shares, uint256 i)
        private view returns (uint256 dy, uint256 fee)
    {
        uint256 _totalSupply = totalSupply;
        uint256[N] memory xp = _xp();

        uint256 d0 = _getD(xp);
        uint256 d1 = d0 - (d0 * shares) / _totalSupply;  // 取出份额后的 D

        uint256 y0 = _getYD(i, xp, d1);
        uint256 dy0 = (xp[i] - y0) / multipliers[i];

        // 计算不平衡费
        uint256 dx;
        for (uint256 j; j < N; ++j) {
            if (j == i) {
                dx = (xp[j] * d1) / d0 - y0;
            } else {
                dx = xp[j] - (xp[j] * d1) / d0;
            }
            xp[j] -= (LIQUIDITY_FEE * dx) / FEE_DENOMINATOR;
        }

        uint256 y1 = _getYD(i, xp, d1);
        dy = (xp[i] - y1 - 1) / multipliers[i];
        fee = dy0 - dy;
    }

    // _getYD 函数：已知 D 计算代币 i 的新余额
    function _getYD(uint256 i, uint256[N] memory xp, uint256 d)
        private pure returns (uint256)
    {
        uint256 a = A * N;
        uint256 s;
        uint256 c = d;

        for (uint256 k; k < N; ++k) {
            if (k != i) {
                _x = xp[k];
                s += _x;
                c = (c * d) / (N * _x);
            }
        }
        c = (c * d) / (N * a);
        uint256 b = s + d / a;

        uint256 y_prev;
        uint256 y = d;
        for (uint256 _i; _i < 255; ++_i) {
            y_prev = y;
            y = (y * y + c) / (2 * y + b - d);
            if (Math.abs(y, y_prev) <= 1) { return y; }
        }
        revert("y didn't converge");
    }
}

interface IERC20 { /* 标准 ERC20 接口 */ }
```

## 三种 AMM 对比

| 特性 | CSAMM | CPAMM | StableSwap |
|------|-------|-------|------------|
| 不变量 | `∑x = k` | `∏x = k` | 混合曲线 |
| 滑点 | 零 | 中等 | 1:1 附近极低 |
| 适用场景 | 稳定币对 | 波动资产 | 锚定资产池 |
| 算法复杂度 | O(1) | O(1) | O(n²) 牛顿迭代 |
