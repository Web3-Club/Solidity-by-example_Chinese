# Uniswap V2 添加与移除流动性

对应英文原页：https://solidity-by-example.org/defi/uniswap-v2-add-remove-liquidity

### 添加 / 移除流动性

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

contract UniswapV2AddLiquidity {
    address private constant FACTORY =
        0x5C69bEe701ef814a2B6a3EDD4B1652CB9cc5aA6f;
    address private constant ROUTER = 0x7a250d5630B4cF539739dF2C5dAcb4c659F2488D;
    address private constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address private constant USDT = 0xdAC17F958D2ee523a2206206994597C13D831ec7;

    function addLiquidity(
        address _tokenA,
        address _tokenB,
        uint256 _amountA,
        uint256 _amountB
    ) external {
        safeTransferFrom(IERC20(_tokenA), msg.sender, address(this), _amountA);
        safeTransferFrom(IERC20(_tokenB), msg.sender, address(this), _amountB);

        safeApprove(IERC20(_tokenA), ROUTER, _amountA);
        safeApprove(IERC20(_tokenB), ROUTER, _amountB);

        (uint256 amountA, uint256 amountB, uint256 liquidity) = IUniswapV2Router(
            ROUTER
        ).addLiquidity(
            _tokenA,
            _tokenB,
            _amountA,
            _amountB,
            1,
            1,
            address(this),
            block.timestamp
        );
    }

    function removeLiquidity(address _tokenA, address _tokenB) external {
        address pair = IUniswapV2Factory(FACTORY).getPair(_tokenA, _tokenB);

        uint256 liquidity = IERC20(pair).balanceOf(address(this));
        safeApprove(IERC20(pair), ROUTER, liquidity);

        (uint256 amountA, uint256 amountB) = IUniswapV2Router(ROUTER)
            .removeLiquidity(
            _tokenA, _tokenB, liquidity, 1, 1, address(this), block.timestamp
        );
    }

    /**
     * @dev transferFrom 可能返回 bool，也可能不返回。
     * ERC-20 规范要求返回 bool，但有些代币并不遵循该规范。
     * 需要检查返回数据为空或为 true。
     */
    function safeTransferFrom(
        IERC20 token,
        address sender,
        address recipient,
        uint256 amount
    ) internal {
        (bool success, bytes memory returnData) = address(token).call(
            abi.encodeCall(IERC20.transferFrom, (sender, recipient, amount))
        );
        require(
            success
                && (returnData.length == 0 || abi.decode(returnData, (bool))),
            "Transfer from fail"
        );
    }

    /**
     * @dev approve 可能返回 bool，也可能不返回。
     * ERC-20 规范要求返回 bool，但有些代币并不遵循该规范。
     * 需要检查返回数据为空或为 true。
     */
    function safeApprove(IERC20 token, address spender, uint256 amount)
        internal
    {
        (bool success, bytes memory returnData) = address(token).call(
            abi.encodeCall(IERC20.approve, (spender, amount))
        );
        require(
            success
                && (returnData.length == 0 || abi.decode(returnData, (bool))),
            "Approve fail"
        );
    }
}

interface IUniswapV2Router {
    function addLiquidity(
        address tokenA,
        address tokenB,
        uint256 amountADesired,
        uint256 amountBDesired,
        uint256 amountAMin,
        uint256 amountBMin,
        address to,
        uint256 deadline
    ) external returns (uint256 amountA, uint256 amountB, uint256 liquidity);

    function removeLiquidity(
        address tokenA,
        address tokenB,
        uint256 liquidity,
        uint256 amountAMin,
        uint256 amountBMin,
        address to,
        uint256 deadline
    ) external returns (uint256 amountA, uint256 amountB);
}

interface IUniswapV2Factory {
    function getPair(address token0, address token1)
        external
        view
        returns (address);
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

### 使用 Foundry 测试

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

import {Test} from "forge-std/Test.sol";
import
    "../../../src/defi/uniswap-v2-add-remove-liquidity/UniswapV2Liquidity.sol";

IERC20 constant WETH = IERC20(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2);
IERC20 constant USDT = IERC20(0xdAC17F958D2ee523a2206206994597C13D831ec7);
IERC20 constant PAIR = IERC20(0x0d4a11d5EEaaC28EC3F61d100daF4d40471f1852);

contract UniswapV2AddLiquidityTest is Test {
    UniswapV2AddLiquidity private uni = new UniswapV2AddLiquidity();

    // 向 Uniswap 添加 WETH/USDT 流动性
    function testAddLiquidity() public {
        // 向本合约发放测试用 USDT 和 WETH
        deal(address(USDT), address(this), 1e6 * 1e6);
        assertEq(
            USDT.balanceOf(address(this)), 1e6 * 1e6, "USDT balance incorrect"
        );
        deal(address(WETH), address(this), 1e6 * 1e18);
        assertEq(
            WETH.balanceOf(address(this)), 1e6 * 1e18, "WETH balance incorrect"
        );

        // 授权 uni 进行转账
        safeApprove(WETH, address(uni), 1e64);
        safeApprove(USDT, address(uni), 1e64);

        uni.addLiquidity(address(WETH), address(USDT), 1 * 1e18, 3000.05 * 1e6);

        assertGt(PAIR.balanceOf(address(uni)), 0, "pair balance 0");
    }

    // 从 Uniswap 移除 WETH/USDT 流动性
    function testRemoveLiquidity() public {
        // 向 uni 发放 LP 代币
        deal(address(PAIR), address(uni), 1e10);
        assertEq(PAIR.balanceOf(address(uni)), 1e10, "LP tokens balance = 0");
        assertEq(USDT.balanceOf(address(uni)), 0, "USDT balance non-zero");
        assertEq(WETH.balanceOf(address(uni)), 0, "WETH balance non-zero");

        uni.removeLiquidity(address(WETH), address(USDT));

        assertEq(PAIR.balanceOf(address(uni)), 0, "LP tokens balance != 0");
        assertGt(USDT.balanceOf(address(uni)), 0, "USDT balance = 0");
        assertGt(WETH.balanceOf(address(uni)), 0, "WETH balance = 0");
    }

    /**
     * @dev transferFrom 可能返回 bool，也可能不返回。
     * ERC-20 规范要求返回 bool，但有些代币并不遵循该规范。
     * 需要检查返回数据为空或为 true。
     */
    function safeTransferFrom(
        IERC20 token,
        address sender,
        address recipient,
        uint256 amount
    ) internal {
        (bool success, bytes memory returnData) = address(token).call(
            abi.encodeCall(IERC20.transferFrom, (sender, recipient, amount))
        );
        require(
            success
                && (returnData.length == 0 || abi.decode(returnData, (bool))),
            "Transfer from fail"
        );
    }

    /**
     * @dev approve 可能返回 bool，也可能不返回。
     * ERC-20 规范要求返回 bool，但有些代币并不遵循该规范。
     * 需要检查返回数据为空或为 true。
     */
    function safeApprove(IERC20 token, address spender, uint256 amount)
        internal
    {
        (bool success, bytes memory returnData) = address(token).call(
            abi.encodeCall(IERC20.approve, (spender, amount))
        );
        require(
            success
                && (returnData.length == 0 || abi.decode(returnData, (bool))),
            "Approve fail"
        );
    }
}

```

---
## 关注我们
[Yanbo的Twitter](https://twitter.com/YanboOfficial)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)

加入Web3Club 官方讨论群：YanboTravelAllWorld（微信号）

[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
