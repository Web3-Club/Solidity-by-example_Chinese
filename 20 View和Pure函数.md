# View 和 Pure 函数

对应英文原页：https://solidity-by-example.org/view-and-pure-functions

Getter 函数可以声明为 `view` 或 `pure`。

`View` 函数声明不会修改状态。

`Pure` 函数声明既不会修改也不会读取状态变量。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

contract ViewAndPure {
    uint256 public x = 1;

    // 承诺不修改状态。
    function addToX(uint256 y) public view returns (uint256) {
        return x + y;
    }

    // 承诺既不修改也不读取状态。
    function add(uint256 i, uint256 j) public pure returns (uint256) {
        return i + j;
    }
}
```

---
## 关注我们
[Yanbo的Twitter](https://x.com/Yanbo2004)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)


[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
