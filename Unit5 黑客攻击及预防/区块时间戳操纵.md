# 区块时间戳操纵

对应英文原页：https://solidity-by-example.org/hacks/block-timestamp-manipulation

## 漏洞

矿工可以在以下约束下操纵 `block.timestamp`

- 它不能早于父区块的时间
- 它不能过于未来

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

/*
Roulette 是一个游戏：如果你能在特定时机提交交易，
就可以赢得合约中的全部 Ether。
玩家需要发送 10 Ether，若 block.timestamp % 15 == 0 则获胜。
*/

/*
1. 部署 Roulette 并放入 10 Ether
2. Eve 运行一个强大的矿工，可以操纵区块时间戳。
3. Eve 把 block.timestamp 设为一个可被 15 整除的未来时间，
   并找到目标区块哈希。
4. Eve 的区块成功被包含进链中，Eve 赢得了
   Roulette 游戏。
*/

contract Roulette {
    uint256 public pastBlockTime;

    constructor() payable {}

    function spin() external payable {
        require(msg.value == 10 ether); // 必须发送 10 ether 才能玩
        require(block.timestamp != pastBlockTime); // 每个区块只能有 1 笔交易

        pastBlockTime = block.timestamp;

        if (block.timestamp % 15 == 0) {
            (bool sent,) = msg.sender.call{value: address(this).balance}("");
            require(sent, "Failed to send Ether");
        }
    }
}
```

## 预防措施

- 不要使用 `block.timestamp` 作为熵源和随机数

---
## 关注我们
[Yanbo的Twitter](https://x.com/Yanbo2004)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)


[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
