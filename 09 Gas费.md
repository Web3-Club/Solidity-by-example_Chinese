# Gas 与 Gas Price

对应英文原页：https://solidity-by-example.org/gas

### 一笔交易需要支付多少 `ether`？

你需要支付 `gas spent * gas price` 数量的 `ether`，其中

- `gas` 是计算单位
- `gas spent` 是交易中使用的 `gas` 总量
- `gas price` 是你愿意为每单位 `gas` 支付的 `ether` 数量

gas price 更高的交易会优先被打包进区块。

未使用的 gas 会被退还。

### Gas 上限（Gas Limit）

你可以花费的 gas 有两个上限

- `gas limit`（你为交易设置的、愿意使用的最大 gas 数量）
- `block gas limit`（网络设置的、一个区块中允许的最大 gas 数量）

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

contract Gas {
    uint256 public i = 0;

    // 用尽你发送的全部 gas 会导致交易失败。
    // 状态更改会被撤销。
    // 已花费的 gas 不会退还。
    function forever() public {
        // 这里我们运行一个循环，直到用尽全部 gas
        // 并且交易失败
        while (true) {
            i += 1;
        }
    }
}
```

---
## 关注我们
[Yanbo的Twitter](https://twitter.com/YanboOfficial)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)

加入Web3Club 官方讨论群：YanboTravelAllWorld（微信号）

[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
