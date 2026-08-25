# 汇编变量

对应英文原页：https://solidity-by-example.org/assembly-variable

如何在 `assembly` 中声明变量的示例

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

contract AssemblyVariable {
    function yul_let() public pure returns (uint256 z) {
        assembly {
            // 汇编所用的语言叫做 Yul
            // 局部变量
            let x := 123
            z := 456
        }
    }
}
```

---
## 关注我们
[Yanbo的Twitter](https://x.com/Yanbo2004)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)


[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
