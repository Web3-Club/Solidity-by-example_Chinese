# 第一个应用

对应英文原页：https://solidity-by-example.org/first-app

下面是一个简单的合约，你可以获取、增加和减少存储在该合约中的计数值。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

contract Counter {
    uint256 public count;

    // 获取当前计数值的函数
    function get() public view returns (uint256) {
        return count;
    }

    // 将计数值加 1 的函数
    function inc() public {
        count += 1;
    }

    // 将计数值减 1 的函数
    function dec() public {
        // 如果 count = 0，此函数会失败
        count -= 1;
    }
}
```

---
## 关注我们
[Yanbo的Twitter](https://x.com/Yanbo2004)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)


[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
