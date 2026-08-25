# 继承

对应英文原页：https://solidity-by-example.org/inheritance

Solidity 支持多重继承。合约可以使用 `is` 关键字继承其他合约。

将被子合约重写（override）的函数必须声明为 `virtual`。

将要重写父函数的函数必须使用关键字 `override`。

继承顺序很重要。

必须按从「最基础」（most base-like）到「最派生」（most derived）的顺序列出父合约。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

/* 继承关系图
    A
   / \
  B   C
 / \ /
F  D,E

*/

contract A {
    function foo() public pure virtual returns (string memory) {
        return "A";
    }
}

// 合约使用关键字 'is' 继承其他合约。
contract B is A {
    // 重写 A.foo()
    function foo() public pure virtual override returns (string memory) {
        return "B";
    }
}

contract C is A {
    // 重写 A.foo()
    function foo() public pure virtual override returns (string memory) {
        return "C";
    }
}

// 合约可以从多个父合约继承。
// 当调用的函数在不同合约中被多次定义时，
// 会从右到左、以深度优先的方式搜索父合约。

contract D is B, C {
    // D.foo() 返回 "C"
    // 因为 C 是拥有 foo() 函数的最右侧父合约
    function foo() public pure override(B, C) returns (string memory) {
        return super.foo();
    }
}

contract E is C, B {
    // E.foo() 返回 "B"
    // 因为 B 是拥有 foo() 函数的最右侧父合约
    function foo() public pure override(C, B) returns (string memory) {
        return super.foo();
    }
}

// 继承必须按从「最基础」到「最派生」的顺序排列。
// 交换 A 和 B 的顺序会引发编译错误。
contract F is A, B {
    function foo() public pure override(A, B) returns (string memory) {
        return super.foo();
    }
}
```

---
## 关注我们
[Yanbo的Twitter](https://x.com/Yanbo2004)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)


[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
