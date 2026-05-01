# Uniswap V2 闪电交换

Uniswap V2 的闪电交换（Flash Swap）允许用户"先使用后支付"，可以在不持有本金的情况下进行套利等操作。用户在一次交易中借出代币、使用代币、然后归还本金+手续费。

```solidity
// SPDX-License-Identifier: MIT  // SPDX 许可证标识符：MIT 许可证
pragma solidity ^0.8.26;  // 版本指令，要求使用 Solidity 0.8.26 及以上版本

// IUniswapV2Callee 接口：闪电交换回调接口
interface IUniswapV2Callee {
    function uniswapV2Call(
        address sender, uint256 amount0, uint256 amount1, bytes calldata data
    ) external;
}

// UniswapV2FlashSwap 合约：Uniswap V2 闪电交换实现
contract UniswapV2FlashSwap is IUniswapV2Callee {  // 实现回调接口
    // Uniswap V2 工厂合约地址
    address private constant UNISWAP_V2_FACTORY =
        0x5C69bEe701ef814a2B6a3EDD4B1652CB9cc5aA6f;

    address private constant DAI = 0x6B175474E89094C44Da98b954EedeAC495271d0F;
    address private constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;

    IUniswapV2Factory private constant factory =
        IUniswapV2Factory(UNISWAP_V2_FACTORY);
    IERC20 private constant weth = IERC20(WETH);
    IUniswapV2Pair private immutable pair;  // DAI-WETH 交易对

    uint256 public amountToRepay;  // 需偿还的总金额（本金+手续费）

    constructor() {
        pair = IUniswapV2Pair(factory.getPair(DAI, WETH));  // 获取交易对地址
    }

    // flashSwap 函数：发起闪电交换
    function flashSwap(uint256 wethAmount) external {
        // 编码回调数据（借入的代币和调用者地址）
        bytes memory data = abi.encode(WETH, msg.sender);

        // amount0Out = DAI 输出（0），amount1Out = WETH 输出（借入量）
        pair.swap(0, wethAmount, address(this), data);
    }

    // uniswapV2Call 函数：闪电交换回调（由交易对合约调用）
    function uniswapV2Call(
        address sender,
        uint256 amount0,
        uint256 amount1,
        bytes calldata data
    ) external {
        require(msg.sender == address(pair), "not pair");
        require(sender == address(this), "not sender");

        (address tokenBorrow, address caller) =
            abi.decode(data, (address, address));

        // 在此处编写套利或其他自定义逻辑
        require(tokenBorrow == WETH, "token borrow != WETH");

        // 计算手续费（约 0.3%），+1 向上取整
        uint256 fee = (amount1 * 3) / 997 + 1;
        amountToRepay = amount1 + fee;  // 借入量 + 手续费

        // 从调用者转入手续费（调用者需提前授权）
        weth.transferFrom(caller, address(this), fee);

        // 归还本金 + 手续费给交易对
        weth.transfer(address(pair), amountToRepay);
    }
}

interface IUniswapV2Pair {
    function swap(uint256 amount0Out, uint256 amount1Out, address to, bytes calldata data) external;
}
interface IUniswapV2Factory {
    function getPair(address tokenA, address tokenB) external view returns (address pair);
}
interface IERC20 { /* 标准 ERC20 接口 */ }
interface IWETH is IERC20 {
    function deposit() external payable;
    function withdraw(uint256 amount) external;
}
```

## 闪电交换 vs 闪电贷

| 特性 | Uniswap V2 闪电交换 | 传统闪电贷 |
|------|-------------------|----------|
| 调用入口 | `pair.swap(0, amount, to, data)` | 借贷池合约 |
| 手续费 | 0.3% Uniswap 手续费 | 平台费（如 Aave 0.09%）|
| 回调函数 | `uniswapV2Call` | 合约特定函数 |
