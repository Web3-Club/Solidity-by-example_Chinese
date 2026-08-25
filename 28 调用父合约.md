# 调用父合约

对应英文原页：https://solidity-by-example.org/super

可以直接调用父合约，也可以使用关键字 `super`。

使用关键字 `super` 时，所有直接父合约都会被调用。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

/* 继承树
   A
 /  \
B   C
 \ /
  D
*/

contract A {
    // 这称为事件。你可以在函数中发出事件，
    // 它们会被记录到交易日志中。
    // 在本例中，这有助于追踪函数调用。
    event Log(string message);

    function foo() public virtual {
        emit Log("A.foo called");
    }

    function bar() public virtual {
        emit Log("A.bar called");
    }
}

contract B is A {
    function foo() public virtual override {
        emit Log("B.foo called");
        A.foo();
    }

    function bar() public virtual override {
        emit Log("B.bar called");
        super.bar();
    }
}

contract C is A {
    function foo() public virtual override {
        emit Log("C.foo called");
        A.foo();
    }

    function bar() public virtual override {
        emit Log("C.bar called");
        super.bar();
    }
}

contract D is B, C {
    // 试一试：
    // - 调用 D.foo 并查看交易日志。
    //   尽管 D 继承了 A、B 和 C，它只调用了 C，然后是 A。
    // - 调用 D.bar 并查看交易日志。
    //   D 调用了 C，然后是 B，最后是 A。
    //   尽管 super 被调用了两次（由 B 和 C），它只调用了 A 一次。

    function foo() public override(B, C) {
        super.foo();
    }

    function bar() public override(B, C) {
        super.bar();
    }
}
```

---
## 关注我们
[Yanbo的Twitter](https://x.com/Yanbo2004)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)


[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
