# Uniswap V3 流动性

对应英文原页：https://solidity-by-example.org/defi/uniswap-v3-liquidity

铸造新仓位、收取手续费、增加和减少流动性的示例。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

address constant DAI = 0x6B175474E89094C44Da98b954EedeAC495271d0F;
address constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;

interface IERC721Receiver {
    function onERC721Received(
        address operator,
        address from,
        uint256 tokenId,
        bytes calldata data
    ) external returns (bytes4);
}

contract UniswapV3Liquidity is IERC721Receiver {
    IERC20 private constant dai = IERC20(DAI);
    IWETH private constant weth = IWETH(WETH);

    int24 private constant MIN_TICK = -887272;
    int24 private constant MAX_TICK = -MIN_TICK;
    int24 private constant TICK_SPACING = 60;

    INonfungiblePositionManager public nonfungiblePositionManager =
        INonfungiblePositionManager(0xC36442b4a4522E871399CD717aBDD847Ab11FE88);

    function onERC721Received(
        address operator,
        address from,
        uint256 tokenId,
        bytes calldata
    ) external returns (bytes4) {
        return IERC721Receiver.onERC721Received.selector;
    }

    function mintNewPosition(uint256 amount0ToAdd, uint256 amount1ToAdd)
        external
        returns (
            uint256 tokenId,
            uint128 liquidity,
            uint256 amount0,
            uint256 amount1
        )
    {
        dai.transferFrom(msg.sender, address(this), amount0ToAdd);
        weth.transferFrom(msg.sender, address(this), amount1ToAdd);

        dai.approve(address(nonfungiblePositionManager), amount0ToAdd);
        weth.approve(address(nonfungiblePositionManager), amount1ToAdd);

        INonfungiblePositionManager.MintParams memory params =
        INonfungiblePositionManager.MintParams({
            token0: DAI,
            token1: WETH,
            fee: 3000,
            tickLower: (MIN_TICK / TICK_SPACING) * TICK_SPACING,
            tickUpper: (MAX_TICK / TICK_SPACING) * TICK_SPACING,
            amount0Desired: amount0ToAdd,
            amount1Desired: amount1ToAdd,
            amount0Min: 0,
            amount1Min: 0,
            recipient: address(this),
            deadline: block.timestamp
        });

        (tokenId, liquidity, amount0, amount1) =
            nonfungiblePositionManager.mint(params);

        if (amount0 < amount0ToAdd) {
            dai.approve(address(nonfungiblePositionManager), 0);
            uint256 refund0 = amount0ToAdd - amount0;
            dai.transfer(msg.sender, refund0);
        }
        if (amount1 < amount1ToAdd) {
            weth.approve(address(nonfungiblePositionManager), 0);
            uint256 refund1 = amount1ToAdd - amount1;
            weth.transfer(msg.sender, refund1);
        }
    }

    function collectAllFees(uint256 tokenId)
        external
        returns (uint256 amount0, uint256 amount1)
    {
        INonfungiblePositionManager.CollectParams memory params =
        INonfungiblePositionManager.CollectParams({
            tokenId: tokenId,
            recipient: address(this),
            amount0Max: type(uint128).max,
            amount1Max: type(uint128).max
        });

        (amount0, amount1) = nonfungiblePositionManager.collect(params);
    }

    function increaseLiquidityCurrentRange(
        uint256 tokenId,
        uint256 amount0ToAdd,
        uint256 amount1ToAdd
    ) external returns (uint128 liquidity, uint256 amount0, uint256 amount1) {
        dai.transferFrom(msg.sender, address(this), amount0ToAdd);
        weth.transferFrom(msg.sender, address(this), amount1ToAdd);

        dai.approve(address(nonfungiblePositionManager), amount0ToAdd);
        weth.approve(address(nonfungiblePositionManager), amount1ToAdd);

        INonfungiblePositionManager.IncreaseLiquidityParams memory params =
        INonfungiblePositionManager.IncreaseLiquidityParams({
            tokenId: tokenId,
            amount0Desired: amount0ToAdd,
            amount1Desired: amount1ToAdd,
            amount0Min: 0,
            amount1Min: 0,
            deadline: block.timestamp
        });

        (liquidity, amount0, amount1) =
            nonfungiblePositionManager.increaseLiquidity(params);
    }

    function decreaseLiquidityCurrentRange(uint256 tokenId, uint128 liquidity)
        external
        returns (uint256 amount0, uint256 amount1)
    {
        INonfungiblePositionManager.DecreaseLiquidityParams memory params =
        INonfungiblePositionManager.DecreaseLiquidityParams({
            tokenId: tokenId,
            liquidity: liquidity,
            amount0Min: 0,
            amount1Min: 0,
            deadline: block.timestamp
        });

        (amount0, amount1) =
            nonfungiblePositionManager.decreaseLiquidity(params);
    }
}

interface INonfungiblePositionManager {
    struct MintParams {
        address token0;
        address token1;
        uint24 fee;
        int24 tickLower;
        int24 tickUpper;
        uint256 amount0Desired;
        uint256 amount1Desired;
        uint256 amount0Min;
        uint256 amount1Min;
        address recipient;
        uint256 deadline;
    }

    function mint(MintParams calldata params)
        external
        payable
        returns (
            uint256 tokenId,
            uint128 liquidity,
            uint256 amount0,
            uint256 amount1
        );

    struct IncreaseLiquidityParams {
        uint256 tokenId;
        uint256 amount0Desired;
        uint256 amount1Desired;
        uint256 amount0Min;
        uint256 amount1Min;
        uint256 deadline;
    }

    function increaseLiquidity(IncreaseLiquidityParams calldata params)
        external
        payable
        returns (uint128 liquidity, uint256 amount0, uint256 amount1);

    struct DecreaseLiquidityParams {
        uint256 tokenId;
        uint128 liquidity;
        uint256 amount0Min;
        uint256 amount1Min;
        uint256 deadline;
    }

    function decreaseLiquidity(DecreaseLiquidityParams calldata params)
        external
        payable
        returns (uint256 amount0, uint256 amount1);

    struct CollectParams {
        uint256 tokenId;
        address recipient;
        uint128 amount0Max;
        uint128 amount1Max;
    }

    function collect(CollectParams calldata params)
        external
        payable
        returns (uint256 amount0, uint256 amount1);
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

interface IWETH is IERC20 {
    function deposit() external payable;
    function withdraw(uint256 amount) external;
}

```

### 使用 Foundry 测试

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

import {Test, console2} from "forge-std/Test.sol";
import "../../../src/defi/uniswap-v3-liquidity/UniswapV3Liquidity.sol";

contract UniswapV3LiquidityTest is Test {
    IWETH private constant weth = IWETH(WETH);
    IERC20 private constant dai = IERC20(DAI);

    address private constant DAI_WHALE =
        0xe81D6f03028107A20DBc83176DA82aE8099E9C42;

    UniswapV3Liquidity private uni = new UniswapV3Liquidity();

    function setUp() public {
        vm.prank(DAI_WHALE);
        dai.transfer(address(this), 20 * 1e18);

        weth.deposit{value: 2 * 1e18}();

        dai.approve(address(uni), 20 * 1e18);
        weth.approve(address(uni), 2 * 1e18);
    }

    function testLiquidity() public {
        // 跟踪总流动性
        uint128 liquidity;

        // 铸造新仓位
        uint256 daiAmount = 10 * 1e18;
        uint256 wethAmount = 1e18;

        (
            uint256 tokenId,
            uint128 liquidityDelta,
            uint256 amount0,
            uint256 amount1
        ) = uni.mintNewPosition(daiAmount, wethAmount);
        liquidity += liquidityDelta;

        console2.log("--- Mint new position ---");
        console2.log("token id", tokenId);
        console2.log("liquidity", liquidity);
        console2.log("amount 0", amount0);
        console2.log("amount 1", amount1);

        // 收取手续费
        (uint256 fee0, uint256 fee1) = uni.collectAllFees(tokenId);

        console2.log("--- Collect fees ---");
        console2.log("fee 0", fee0);
        console2.log("fee 1", fee1);

        // 增加流动性
        uint256 daiAmountToAdd = 5 * 1e18;
        uint256 wethAmountToAdd = 0.5 * 1e18;

        (liquidityDelta, amount0, amount1) = uni.increaseLiquidityCurrentRange(
            tokenId, daiAmountToAdd, wethAmountToAdd
        );
        liquidity += liquidityDelta;

        console2.log("--- Increase liquidity ---");
        console2.log("liquidity", liquidity);
        console2.log("amount 0", amount0);
        console2.log("amount 1", amount1);

        // 减少流动性
        (amount0, amount1) =
            uni.decreaseLiquidityCurrentRange(tokenId, liquidity);
        console2.log("--- Decrease liquidity ---");
        console2.log("amount 0", amount0);
        console2.log("amount 1", amount1);
    }
}

```

### 链接

[Uniswap V3](https://docs.uniswap.org/protocol/guides/providing-liquidity/setting-up)

[Foundry](https://github.com/foundry-rs/foundry)

[Uniswap V3 Foundry 示例](https://github.com/t4sk/defi-notes)

---
## 关注我们
[Yanbo的Twitter](https://twitter.com/YanboOfficial)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)

加入Web3Club 官方讨论群：YanboTravelAllWorld（微信号）

[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
