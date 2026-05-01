# Uniswap V2 最优单边供给

当用户只持有交易对中的一种代币时，可通过计算最优交换量，将部分代币先兑换为另一种代币，再以最优比例添加流动性。

```solidity
// SPDX-License-Identifier: MIT  // SPDX 许可证标识符：MIT 许可证
pragma solidity ^0.8.26;  // 版本指令，要求使用 Solidity 0.8.26 及以上版本

// TestUniswapOptimalOneSidedSupply 合约：单边最优添加流动性
contract TestUniswapOptimalOneSidedSupply {  // 定义最优单边供给合约
    address private constant FACTORY = 0x5C69bEe701ef814a2B6a3EDD4B1652CB9cc5aA6f;
    address private constant ROUTER = 0x7a250d5630B4cF539739dF2C5dAcb4c659F2488D;
    address private constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;

    // sqrt 函数：牛顿迭代法求平方根
    function sqrt(uint256 y) private pure returns (uint256 z) {
        if (y > 3) {
            z = y;
            uint256 x = y / 2 + 1;
            while (x < z) {
                z = x;
                x = (y / x + x) / 2;
            }
        } else if (y != 0) {
            z = 1;
        }
    }

    // getSwapAmount 函数：计算最优交换量
    // s = 最优交换量
    // r = 代币 a 的储备量
    // a = 用户当前持有代币 a 的数量（尚未添加到储备）
    // f = 手续费百分比 (0.3%)
    // s = (sqrt(((2 - f)r)^2 + 4(1 - f)ar) - (2 - f)r) / (2(1 - f))
    function getSwapAmount(uint256 r, uint256 a)
        public
        pure
        returns (uint256)
    {
        // 代入 f = 0.003 化简后的常量
        return (sqrt(r * (r * 3988009 + a * 3988000)) - r * 1997) / 1994;
    }

    // zap 函数：一键最优单边添加流动性
    // 步骤：1. 交换最优数量的 tokenA → tokenB
    //       2. 将剩余的两种代币添加流动性
    function zap(address _tokenA, address _tokenB, uint256 _amountA) external {
        require(_tokenA == WETH || _tokenB == WETH, "!weth");

        IERC20(_tokenA).transferFrom(msg.sender, address(this), _amountA);

        address pair = IUniswapV2Factory(FACTORY).getPair(_tokenA, _tokenB);
        (uint256 reserve0, uint256 reserve1,) =
            IUniswapV2Pair(pair).getReserves();

        uint256 swapAmount;
        if (IUniswapV2Pair(pair).token0() == _tokenA) {
            swapAmount = getSwapAmount(reserve0, _amountA);  // 用 reserve0 计算
        } else {
            swapAmount = getSwapAmount(reserve1, _amountA);  // 用 reserve1 计算
        }

        _swap(_tokenA, _tokenB, swapAmount);      // 步骤 1: 交换
        _addLiquidity(_tokenA, _tokenB);          // 步骤 2: 添加流动性
    }

    function _swap(address _from, address _to, uint256 _amount) internal {
        IERC20(_from).approve(ROUTER, _amount);
        address[] memory path = new address[](2);
        path[0] = _from;
        path[1] = _to;
        IUniswapV2Router(ROUTER).swapExactTokensForTokens(
            _amount, 1, path, address(this), block.timestamp
        );
    }

    function _addLiquidity(address _tokenA, address _tokenB) internal {
        uint256 balA = IERC20(_tokenA).balanceOf(address(this));
        uint256 balB = IERC20(_tokenB).balanceOf(address(this));
        IERC20(_tokenA).approve(ROUTER, balA);
        IERC20(_tokenB).approve(ROUTER, balB);
        IUniswapV2Router(ROUTER).addLiquidity(
            _tokenA, _tokenB, balA, balB, 0, 0, address(this), block.timestamp
        );
    }
}

interface IUniswapV2Router {
    function addLiquidity(address, address, uint256, uint256, uint256, uint256, address, uint256)
        external returns (uint256, uint256, uint256);
    function swapExactTokensForTokens(uint256, uint256, address[] calldata, address, uint256)
        external returns (uint256[] memory);
}
interface IUniswapV2Factory {
    function getPair(address, address) external view returns (address);
}
interface IUniswapV2Pair {
    function token0() external view returns (address);
    function token1() external view returns (address);
    function getReserves() external view returns (uint112, uint112, uint32);
}
interface IERC20 { /* 标准 ERC20 接口 */ }
```

## 最优交换量推导

恒定乘积 AMM 中，若只持有代币 A（数量 a），要最大化添加的流动性，需要找到最优的交换量 s：

```
s = (sqrt(((2-f)r)² + 4(1-f)ar) - (2-f)r) / (2(1-f))
```

其中 r 是代币 A 的储备量，f = 0.003（0.3% 手续费）。
