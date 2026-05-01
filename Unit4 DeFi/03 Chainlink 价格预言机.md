# Chainlink 价格预言机

通过 Chainlink 去中心化预言机网络获取链上资产价格。本例演示如何从 Chainlink 的 ETH/USD 价格馈送合约读取最新价格。

```solidity
// SPDX-License-Identifier: MIT  // SPDX 许可证标识符：MIT 许可证
pragma solidity ^0.8.26;  // 版本指令，要求使用 Solidity 0.8.26 及以上版本

// ChainlinkPriceOracle 合约：从 Chainlink 获取 ETH/USD 价格
contract ChainlinkPriceOracle {  // 定义 Chainlink 价格预言机合约
    AggregatorV3Interface internal priceFeed;  // Chainlink 聚合器接口

    constructor() {
        // ETH / USD 价格馈送合约地址（以太坊主网）
        priceFeed =
            AggregatorV3Interface(0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419);
    }

    // getLatestPrice 函数：获取最新的 ETH/USD 价格
    function getLatestPrice() public view returns (int256) {  // 返回 int256 价格
        (
            uint80 roundID,        // 当前轮次 ID
            int256 price,          // 价格（放大 10^8 倍）
            uint256 startedAt,     // 本轮开始时间戳
            uint256 timeStamp,     // 本轮更新时间戳
            uint80 answeredInRound // 回答时所在轮次
        ) = priceFeed.latestRoundData();  // 获取最新轮次数据
        // ETH / USD 价格按 10^8 倍放大，除以 1e8 得到实际价格
        return price / 1e8;  // 价格（美元，整数）
    }
}

// AggregatorV3Interface 接口：Chainlink 聚合器 V3 标准接口
interface AggregatorV3Interface {  // 定义聚合器接口
    // latestRoundData 函数：获取最新一轮的价格数据
    function latestRoundData()
        external
        view
        returns (
            uint80 roundId,             // 轮次 ID
            int256 answer,              // 价格答案
            uint256 startedAt,          // 开始时间
            uint256 updatedAt,          // 更新时间
            uint80 answeredInRound      // 回答轮次
        );
}
```

## 注意事项

- Chainlink 价格标度因喂价对而异（例如 ETH/USD 为 8 位小数）
- 价格是 `int256` 类型，可能为负数（极少情况）
- 不同网络的合约地址不同，需从 [Chainlink 文档](https://docs.chain.link/data-feeds/price-feeds/addresses) 获取
