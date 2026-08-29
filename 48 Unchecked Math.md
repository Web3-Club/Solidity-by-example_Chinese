# Unchecked Math

对应英文原页：https://solidity-by-example.org/unchecked-math

在 Solidity 0.8 中，数值溢出（overflow）和下溢（underflow）会抛出错误。可以使用 `unchecked` 关闭该检查。

关闭溢出 / 下溢检查可以节省 gas。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

contract UncheckedMath {
    function add(uint256 x, uint256 y) external pure returns (uint256) {
        // 22291 gas
        // return x + y;

        // 22103 gas
        unchecked {
            return x + y;
        }
    }

    function sub(uint256 x, uint256 y) external pure returns (uint256) {
        // 22329 gas
        // return x - y;

        // 22147 gas
        unchecked {
            return x - y;
        }
    }

    function sumOfCubes(uint256 x, uint256 y) external pure returns (uint256) {
        // 把复杂数学逻辑包在 unchecked 中
        unchecked {
            uint256 x3 = x * x * x;
            uint256 y3 = y * y * y;

            return x3 + y3;
        }
    }
}
```

---
## 关注我们
[Yanbo的Twitter](https://x.com/Yanbo2004)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)


[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
