# 以太币和 Wei

对应英文原页：https://solidity-by-example.org/ether-units

交易使用 `ether` 支付。

类似于 1 美元等于 100 美分，1 个 `ether` 等于 10<sup>18</sup> `wei`。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

contract EtherUnits {
    uint256 public oneWei = 1 wei;
    // 1 wei 等于 1
    bool public isOneWei = (oneWei == 1);

    uint256 public oneGwei = 1 gwei;
    // 1 gwei 等于 10^9 wei
    bool public isOneGwei = (oneGwei == 1e9);

    uint256 public oneEther = 1 ether;
    // 1 ether 等于 10^18 wei
    bool public isOneEther = (oneEther == 1e18);
}
```

---
## 关注我们
[Yanbo的Twitter](https://twitter.com/YanboOfficial)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)

加入Web3Club 官方讨论群：YanboTravelAllWorld（微信号）

[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
