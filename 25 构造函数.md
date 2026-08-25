# 构造函数

对应英文原页：https://solidity-by-example.org/constructor

`constructor`（构造函数）是在合约创建时执行的可选函数。

下面是如何向 `constructors` 传递参数的示例。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

// 基合约 X
contract X {
    string public name;

    constructor(string memory _name) {
        name = _name;
    }
}

// 基合约 Y
contract Y {
    string public text;

    constructor(string memory _text) {
        text = _text;
    }
}

// 有两种方式用参数初始化父合约。

// 在继承列表中传入参数。
contract B is X("Input to X"), Y("Input to Y") {}

contract C is X, Y {
    // 在构造函数中传入参数，
    // 类似于函数修饰符。
    constructor(string memory _name, string memory _text) X(_name) Y(_text) {}
}

// 无论子合约构造函数中列出父合约的顺序如何，
// 父构造函数始终按继承顺序被调用。

// 构造函数的调用顺序：
// 1. X
// 2. Y
// 3. D
contract D is X, Y {
    constructor() X("X was called") Y("Y was called") {}
}

// 构造函数的调用顺序：
// 1. X
// 2. Y
// 3. E
contract E is X, Y {
    constructor() Y("Y was called") X("X was called") {}
}
```

---
## 关注我们
[Yanbo的Twitter](https://x.com/Yanbo2004)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)


[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
