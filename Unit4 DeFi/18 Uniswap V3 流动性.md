# Uniswap V3 流动性管理

Uniswap V3 使用 NFT 代表流动性仓位（Position），支持在自定义的价格区间内提供集中流动性。通过 `NonfungiblePositionManager` 进行铸造、增加、减少和收取手续费。

```solidity
// SPDX-License-Identifier: MIT  // SPDX 许可证标识符：MIT 许可证
pragma solidity ^0.8.26;  // 版本指令，要求使用 Solidity 0.8.26 及以上版本

address constant DAI = 0x6B175474E89094C44Da98b954EedeAC495271d0F;
address constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;

// UniswapV3Liquidity 合约：V3 流动性管理
contract UniswapV3Liquidity is IERC721Receiver {  // 实现 ERC721 接收器
    IERC20 private constant dai = IERC20(DAI);
    IWETH private constant weth = IWETH(WETH);

    int24 private constant MIN_TICK = -887272;  // 最小 tick
    int24 private constant MAX_TICK = -MIN_TICK; // 最大 tick（对称）
    int24 private constant TICK_SPACING = 60;    // 费率 0.3% 的 tick 间距

    // NonfungiblePositionManager 合约地址（主网）
    INonfungiblePositionManager public nonfungiblePositionManager =
        INonfungiblePositionManager(0xC36442b4a4522E871399CD717aBDD847Ab11FE88);

    // 接收 NFT 的回调（返回 selector 表示支持）
    function onERC721Received(address, address, uint256, bytes calldata)
        external pure returns (bytes4) {
        return IERC721Receiver.onERC721Received.selector;
    }

    // mintNewPosition 函数：创建新的流动性仓位
    function mintNewPosition(uint256 amount0ToAdd, uint256 amount1ToAdd)
        external
        returns (uint256 tokenId, uint128 liquidity, uint256 amount0, uint256 amount1)
    {
        dai.transferFrom(msg.sender, address(this), amount0ToAdd);
        weth.transferFrom(msg.sender, address(this), amount1ToAdd);
        dai.approve(address(nonfungiblePositionManager), amount0ToAdd);
        weth.approve(address(nonfungiblePositionManager), amount1ToAdd);

        INonfungiblePositionManager.MintParams memory params =
        INonfungiblePositionManager.MintParams({
            token0: DAI,
            token1: WETH,
            fee: 3000,  // 0.3% 费率等级
            // 设置全范围 tick（对齐到 tickSpacing）
            tickLower: (MIN_TICK / TICK_SPACING) * TICK_SPACING,
            tickUpper: (MAX_TICK / TICK_SPACING) * TICK_SPACING,
            amount0Desired: amount0ToAdd, amount1Desired: amount1ToAdd,
            amount0Min: 0, amount1Min: 0,
            recipient: address(this),
            deadline: block.timestamp
        });

        (tokenId, liquidity, amount0, amount1) =
            nonfungiblePositionManager.mint(params);

        // 退还多余代币
        _refundIfNeeded(amount0ToAdd, amount0, amount1ToAdd, amount1);
    }

    // collectAllFees 函数：收集仓位的手续费收益
    function collectAllFees(uint256 tokenId)
        external
        returns (uint256 amount0, uint256 amount1)
    {
        INonfungiblePositionManager.CollectParams memory params =
        INonfungiblePositionManager.CollectParams({
            tokenId: tokenId,
            recipient: address(this),  // 手续费收到本合约
            amount0Max: type(uint128).max,  // 收集全部手续费
            amount1Max: type(uint128).max
        });
        (amount0, amount1) = nonfungiblePositionManager.collect(params);
    }

    // increaseLiquidityCurrentRange 函数：为已有仓位增加流动性
    function increaseLiquidityCurrentRange(
        uint256 tokenId, uint256 amount0ToAdd, uint256 amount1ToAdd
    ) external returns (uint128 liquidity, uint256 amount0, uint256 amount1) {
        dai.transferFrom(msg.sender, address(this), amount0ToAdd);
        weth.transferFrom(msg.sender, address(this), amount1ToAdd);
        dai.approve(address(nonfungiblePositionManager), amount0ToAdd);
        weth.approve(address(nonfungiblePositionManager), amount1ToAdd);

        INonfungiblePositionManager.IncreaseLiquidityParams memory params =
        INonfungiblePositionManager.IncreaseLiquidityParams({
            tokenId: tokenId,
            amount0Desired: amount0ToAdd, amount1Desired: amount1ToAdd,
            amount0Min: 0, amount1Min: 0,
            deadline: block.timestamp
        });
        (liquidity, amount0, amount1) =
            nonfungiblePositionManager.increaseLiquidity(params);
    }

    // decreaseLiquidityCurrentRange 函数：减少仓位的流动性
    function decreaseLiquidityCurrentRange(uint256 tokenId, uint128 liquidity)
        external
        returns (uint256 amount0, uint256 amount1)
    {
        INonfungiblePositionManager.DecreaseLiquidityParams memory params =
        INonfungiblePositionManager.DecreaseLiquidityParams({
            tokenId: tokenId,
            liquidity: liquidity,
            amount0Min: 0, amount1Min: 0,
            deadline: block.timestamp
        });
        (amount0, amount1) =
            nonfungiblePositionManager.decreaseLiquidity(params);
    }

    function _refundIfNeeded(uint256 d0, uint256 a0, uint256 d1, uint256 a1)
        private
    {
        if (a0 < d0) { dai.approve(address(nonfungiblePositionManager), 0); dai.transfer(msg.sender, d0 - a0); }
        if (a1 < d1) { weth.approve(address(nonfungiblePositionManager), 0); weth.transfer(msg.sender, d1 - a1); }
    }
}

// 核心接口
interface INonfungiblePositionManager {
    struct MintParams {
        address token0; address token1; uint24 fee;
        int24 tickLower; int24 tickUpper;
        uint256 amount0Desired; uint256 amount1Desired;
        uint256 amount0Min; uint256 amount1Min;
        address recipient; uint256 deadline;
    }
    function mint(MintParams calldata params) external payable
        returns (uint256 tokenId, uint128 liquidity, uint256 amount0, uint256 amount1);

    struct IncreaseLiquidityParams {
        uint256 tokenId; uint256 amount0Desired; uint256 amount1Desired;
        uint256 amount0Min; uint256 amount1Min; uint256 deadline;
    }
    function increaseLiquidity(IncreaseLiquidityParams calldata params) external payable
        returns (uint128 liquidity, uint256 amount0, uint256 amount1);

    struct DecreaseLiquidityParams {
        uint256 tokenId; uint128 liquidity; uint256 amount0Min; uint256 amount1Min; uint256 deadline;
    }
    function decreaseLiquidity(DecreaseLiquidityParams calldata params) external payable
        returns (uint256 amount0, uint256 amount1);

    struct CollectParams {
        uint256 tokenId; address recipient; uint128 amount0Max; uint128 amount1Max;
    }
    function collect(CollectParams calldata params) external payable
        returns (uint256 amount0, uint256 amount1);
}

interface IERC721Receiver {
    function onERC721Received(address operator, address from, uint256 tokenId, bytes calldata data)
        external returns (bytes4);
}
interface IERC20 { /* 标准 ERC20 接口 */ }
interface IWETH is IERC20 {
    function deposit() external payable;
    function withdraw(uint256 amount) external;
}
```

## V3 核心概念

| 概念 | 说明 |
|------|------|
| **Tick** | 价格的整数表示，`price = 1.0001^tick` |
| **Tick Spacing** | 可用的 tick 间隔（由费率决定：0.05%→10, 0.30%→60, 1.00%→200）|
| **区间流动性** | 仅在 tickLower ~ tickUpper 范围内提供流动性 |
| **NFT 仓位** | 每个仓位是独特的 ERC721 代币，可交易/转移 |
| **手续费自动累积** | 手续费在仓位的 token0 和 token1 中累积，需手动收集 |
