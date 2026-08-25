# 汇编条件语句

对应英文原页：https://solidity-by-example.org/assembly-if

`assembly` 中条件语句的示例

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

contract AssemblyIf {
    function yul_if(uint256 x) public pure returns (uint256 z) {
        assembly {
            // if 条件 = 1 { 代码 }
            // 没有 else
            // if 0 { z := 99 }
            // if 1 { z := 99 }
            if lt(x, 10) { z := 99 }
        }
    }

    function yul_switch(uint256 x) public pure returns (uint256 z) {
        assembly {
            switch x
            case 1 { z := 10 }
            case 2 { z := 20 }
            default { z := 0 }
        }
    }
}
```

---
## 关注我们
[Yanbo的Twitter](https://x.com/Yanbo2004)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)


[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
