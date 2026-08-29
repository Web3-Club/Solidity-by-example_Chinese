# Stable Swap AMM

对应英文原页：https://solidity-by-example.org/defi/stable-swap-amm

Curve 稳定兑换 AMM 的简化版本

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

/*
不变量 —— 交易价格和流动性数量由该方程决定

An^n sum(x_i) + D = ADn^n + D^(n + 1) / (n^n prod(x_i))

主题
0. 牛顿法 x_(n + 1) = x_n - f(x_n) / f'(x_n)
1. 不变量
2. 兑换
   - 计算 Y
   - 计算 D
3. 获取虚拟价格
4. 添加流动性
   - 失衡手续费
5. 移除流动性
6. 移除流动性（单一代币）
   - 计算提取单一代币
   - getYD
TODO: 测试？
*/

library Math {
    function abs(uint256 x, uint256 y) internal pure returns (uint256) {
        return x >= y ? x - y : y - x;
    }
}

contract StableSwap {
    // 代币数量
    uint256 private constant N = 3;
    // 放大系数乘以 N^(N - 1)
    // 值越大，曲线越平坦
    // 值越小，曲线越接近恒定乘积 AMM
    uint256 private constant A = 1000 * (N ** (N - 1));
    // 0.03%
    uint256 private constant SWAP_FEE = 300;
    // 流动性手续费由 2 个约束推导
    // 1. 添加 / 移除流动性后资金池仍平衡时，手续费为 0
    // 2. 在平衡池中兑换，等价于向平衡池先添加再移除流动性
    // 兑换手续费 = 添加流动性手续费 + 移除流动性手续费
    uint256 private constant LIQUIDITY_FEE = (SWAP_FEE * N) / (4 * (N - 1));
    uint256 private constant FEE_DENOMINATOR = 1e6;

    address[N] public tokens;
    // 将每种代币归一化到 18 位小数
    // 例如 —— DAI（18 位小数）、USDC（6 位小数）、USDT（6 位小数）
    uint256[N] private multipliers = [1, 1e12, 1e12];
    uint256[N] public balances;

    // 1 份额 = 1e18，18 位小数
    uint256 private constant DECIMALS = 18;
    uint256 public totalSupply;
    mapping(address => uint256) public balanceOf;

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

    // 返回按精度调整后的余额，调整到 18 位小数
    function _xp() private view returns (uint256[N] memory xp) {
        for (uint256 i; i < N; ++i) {
            xp[i] = balances[i] * multipliers[i];
        }
    }

    /**
     * @notice 计算 D，即完全平衡池中的余额之和
     * 若余额为 x_0, x_1, ... x_(n-1)，则 sum(x_i) = D
     * @param xp 按精度调整后的余额
     * @return D
     */
    function _getD(uint256[N] memory xp) private pure returns (uint256) {
        /*
        用牛顿法计算 D
        -----------------------------
        f(D) = ADn^n + D^(n + 1) / (n^n prod(x_i)) - An^n sum(x_i) - D 
        f'(D) = An^n + (n + 1) D^n / (n^n prod(x_i)) - 1

                     (as + np)D_n
        D_(n+1) = -----------------------
                  (a - 1)D_n + (n + 1)p

        a = An^n
        s = sum(x_i)
        p = (D_n)^(n + 1) / (n^n prod(x_i))
        */
        uint256 a = A * N; // An^n

        uint256 s; // x_0 + x_1 + ... + x_(n-1)
        for (uint256 i; i < N; ++i) {
            s += xp[i];
        }

        // 牛顿法
        // 初始猜测，d <= s
        uint256 d = s;
        uint256 d_prev;
        for (uint256 i; i < 255; ++i) {
            // p = D^(n + 1) / (n^n * x_0 * ... * x_(n-1))
            uint256 p = d;
            for (uint256 j; j < N; ++j) {
                p = (p * d) / (N * xp[j]);
            }
            d_prev = d;
            d = ((a * s + N * p) * d) / ((a - 1) * d + (N + 1) * p);

            if (Math.abs(d, d_prev) <= 1) {
                return d;
            }
        }
        revert("D didn't converge");
    }

    /**
     * @notice 给定代币 i 的新余额，计算代币 j 的新余额
     * @param i 输入代币的索引
     * @param j 输出代币的索引
     * @param x 代币 i 的新余额
     * @param xp 当前按精度调整后的余额
     */
    function _getY(uint256 i, uint256 j, uint256 x, uint256[N] memory xp)
        private
        pure
        returns (uint256)
    {
        /*
        用牛顿法计算 y
        -----------------------------
        y = x_j

        f(y) = y^2 + y(b - D) - c

                    y_n^2 + c
        y_(n+1) = --------------
                   2y_n + b - D

        其中
        s = sum(x_k), k != j
        p = prod(x_k), k != j
        b = s + D / (An^n)
        c = D^(n + 1) / (n^n * p * An^n)
        */
        uint256 a = A * N;
        uint256 d = _getD(xp);
        uint256 s;
        uint256 c = d;

        uint256 _x;
        for (uint256 k; k < N; ++k) {
            if (k == i) {
                _x = x;
            } else if (k == j) {
                continue;
            } else {
                _x = xp[k];
            }

            s += _x;
            c = (c * d) / (N * _x);
        }
        c = (c * d) / (N * a);
        uint256 b = s + d / a;

        // 牛顿法
        uint256 y_prev;
        // 初始猜测，y <= d
        uint256 y = d;
        for (uint256 _i; _i < 255; ++_i) {
            y_prev = y;
            y = (y * y + c) / (2 * y + b - d);
            if (Math.abs(y, y_prev) <= 1) {
                return y;
            }
        }
        revert("y didn't converge");
    }

    /**
     * @notice 给定按精度调整后的余额 xp 和流动性 d，计算代币 i 的新余额
     * @dev 计算 y 的方程与 _getY 相同
     * @param i 要计算新余额的代币索引
     * @param xp 按精度调整后的余额
     * @param d 流动性 d
     * @return 代币 i 的新余额
     */
    function _getYD(uint256 i, uint256[N] memory xp, uint256 d)
        private
        pure
        returns (uint256)
    {
        uint256 a = A * N;
        uint256 s;
        uint256 c = d;

        uint256 _x;
        for (uint256 k; k < N; ++k) {
            if (k != i) {
                _x = xp[k];
            } else {
                continue;
            }

            s += _x;
            c = (c * d) / (N * _x);
        }
        c = (c * d) / (N * a);
        uint256 b = s + d / a;

        // 牛顿法
        uint256 y_prev;
        // 初始猜测，y <= d
        uint256 y = d;
        for (uint256 _i; _i < 255; ++_i) {
            y_prev = y;
            y = (y * y + c) / (2 * y + b - d);
            if (Math.abs(y, y_prev) <= 1) {
                return y;
            }
        }
        revert("y didn't converge");
    }

    // 估算 1 份额的价值
    // 1 份额值多少代币？
    function getVirtualPrice() external view returns (uint256) {
        uint256 d = _getD(_xp());
        uint256 _totalSupply = totalSupply;
        if (_totalSupply > 0) {
            return (d * 10 ** DECIMALS) / _totalSupply;
        }
        return 0;
    }

    /**
     * @notice 用 dx 数量的代币 i 兑换代币 j
     * @param i 输入代币的索引
     * @param j 输出代币的索引
     * @param dx 输入代币数量
     * @param minDy 最少输出代币数量
     */
    function swap(uint256 i, uint256 j, uint256 dx, uint256 minDy)
        external
        returns (uint256 dy)
    {
        require(i != j, "i = j");

        IERC20(tokens[i]).transferFrom(msg.sender, address(this), dx);

        // 计算 dy
        uint256[N] memory xp = _xp();
        uint256 x = xp[i] + dx * multipliers[i];

        uint256 y0 = xp[j];
        uint256 y1 = _getY(i, j, x, xp);
        // 由于 x 增加，y0 必须 >= y1
        // -1 用于向下取整
        dy = (y0 - y1 - 1) / multipliers[j];

        // 从 dy 中扣除手续费
        uint256 fee = (dy * SWAP_FEE) / FEE_DENOMINATOR;
        dy -= fee;
        require(dy >= minDy, "dy < min");

        balances[i] += dx;
        balances[j] -= dy;

        IERC20(tokens[j]).transfer(msg.sender, dy);
    }

    function addLiquidity(uint256[N] calldata amounts, uint256 minShares)
        external
        returns (uint256 shares)
    {
        // 计算当前流动性 d0
        uint256 _totalSupply = totalSupply;
        uint256 d0;
        uint256[N] memory old_xs = _xp();
        if (_totalSupply > 0) {
            d0 = _getD(old_xs);
        }

        // 转入代币
        uint256[N] memory new_xs;
        for (uint256 i; i < N; ++i) {
            uint256 amount = amounts[i];
            if (amount > 0) {
                IERC20(tokens[i]).transferFrom(
                    msg.sender, address(this), amount
                );
                new_xs[i] = old_xs[i] + amount * multipliers[i];
            } else {
                new_xs[i] = old_xs[i];
            }
        }

        // 计算新流动性 d1
        uint256 d1 = _getD(new_xs);
        require(d1 > d0, "liquidity didn't increase");

        // 计入失衡手续费后重新计算 D
        uint256 d2;
        if (_totalSupply > 0) {
            for (uint256 i; i < N; ++i) {
                // TODO: 为什么是 old_xs[i] * d1 / d0？为什么不是 d1 / N？
                uint256 idealBalance = (old_xs[i] * d1) / d0;
                uint256 diff = Math.abs(new_xs[i], idealBalance);
                new_xs[i] -= (LIQUIDITY_FEE * diff) / FEE_DENOMINATOR;
            }

            d2 = _getD(new_xs);
        } else {
            d2 = d1;
        }

        // 更新余额
        for (uint256 i; i < N; ++i) {
            balances[i] += amounts[i];
        }

        // 要铸造的份额 = (d2 - d0) / d0 * total supply
        // d1 >= d2 >= d0
        if (_totalSupply > 0) {
            shares = ((d2 - d0) * _totalSupply) / d0;
        } else {
            shares = d2;
        }
        require(shares >= minShares, "shares < min");
        _mint(msg.sender, shares);
    }

    function removeLiquidity(uint256 shares, uint256[N] calldata minAmountsOut)
        external
        returns (uint256[N] memory amountsOut)
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

    /**
     * @notice 计算销毁份额后可收到的代币 i 数量
     * @param shares 要销毁的份额
     * @param i 要提取的代币索引
     * @return dy 可收到的代币 i 数量
     *         fee 提取手续费。手续费已包含在 dy 中
     */
    function _calcWithdrawOneToken(uint256 shares, uint256 i)
        private
        view
        returns (uint256 dy, uint256 fee)
    {
        uint256 _totalSupply = totalSupply;
        uint256[N] memory xp = _xp();

        // 计算 d0 和 d1
        uint256 d0 = _getD(xp);
        uint256 d1 = d0 - (d0 * shares) / _totalSupply;

        // 计算 D = d1 时 y 的减少量
        uint256 y0 = _getYD(i, xp, d1);
        // d1 <= d0，因此 y 必须 <= xp[i]
        uint256 dy0 = (xp[i] - y0) / multipliers[i];

        // 计算失衡手续费，用手续费更新 xp
        uint256 dx;
        for (uint256 j; j < N; ++j) {
            if (j == i) {
                dx = (xp[j] * d1) / d0 - y0;
            } else {
                // d1 / d0 <= 1
                dx = xp[j] - (xp[j] * d1) / d0;
            }
            xp[j] -= (LIQUIDITY_FEE * dx) / FEE_DENOMINATOR;
        }

        // 用包含失衡手续费的 xp 重新计算 y
        uint256 y1 = _getYD(i, xp, d1);
        // - 1 用于向下取整
        dy = (xp[i] - y1 - 1) / multipliers[i];
        fee = dy0 - dy;
    }

    function calcWithdrawOneToken(uint256 shares, uint256 i)
        external
        view
        returns (uint256 dy, uint256 fee)
    {
        return _calcWithdrawOneToken(shares, i);
    }

    /**
     * @notice 以代币 i 提取流动性
     * @param shares 要销毁的份额
     * @param i 要提取的代币
     * @param minAmountOut 必须提取的代币 i 的最小数量
     */
    function removeLiquidityOneToken(
        uint256 shares,
        uint256 i,
        uint256 minAmountOut
    ) external returns (uint256 amountOut) {
        (amountOut,) = _calcWithdrawOneToken(shares, i);
        require(amountOut >= minAmountOut, "out < min");

        balances[i] -= amountOut;
        _burn(msg.sender, shares);

        IERC20(tokens[i]).transfer(msg.sender, amountOut);
    }
}

interface IERC20 {
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function transfer(address recipient, uint256 amount)
        external
        returns (bool);
    function allowance(address owner, address spender)
        external
        view
        returns (uint256);
    function approve(address spender, uint256 amount) external returns (bool);
    function transferFrom(address sender, address recipient, uint256 amount)
        external
        returns (bool);
}

```

---
## 关注我们
[Yanbo的Twitter](https://x.com/Yanbo2004)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)


[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
