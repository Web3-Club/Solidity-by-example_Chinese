# 继承状态变量的屏蔽

对应英文原页：https://solidity-by-example.org/shadowing-inherited-state-variables

与函数不同，状态变量不能通过在子合约中重新声明来重写。

下面学习如何正确地覆盖继承来的状态变量。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

contract A {
    string public name = "Contract A";

    function getName() public view returns (string memory) {
        return name;
    }
}

// Solidity 0.6 禁止屏蔽（shadowing）
// 这段代码无法编译
// contract B is A {
//     string public name = "Contract B";
// }

contract C is A {
    // 这是覆盖继承状态变量的正确方式。
    constructor() {
        name = "Contract C";
    }

    // C.getName 返回 "Contract C"
}
```

---
## 关注我们
[Yanbo的Twitter](https://x.com/Yanbo2004)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)


[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
