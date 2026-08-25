# 使用外部合约隐藏恶意代码

对应英文原页：https://solidity-by-example.org/hacks/hiding-malicious-code-with-external-contract

## 漏洞

在 Solidity 中，任何地址都可以被转换为特定合约类型，
即使该地址上的合约并不是被转换的那种。

这可以被利用来隐藏恶意代码。让我们看看如何做到。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

/*
假设 Alice 可以看到 Foo 和 Bar 的代码，但看不到 Mal。
对 Alice 来说，显然 Foo.callBar() 会执行 Bar.log() 内部的代码。
然而 Eve 使用 Mal 的地址部署了 Foo，因此调用 Foo.callBar()
实际上会执行 Mal 处的代码。
*/

/*
1. Eve 部署 Mal
2. Eve 使用 Mal 的地址部署 Foo
3. Alice 阅读代码并判断调用是安全的之后，调用 Foo.callBar()。
4. 虽然 Alice 预期会执行 Bar.log()，但实际执行的是 Mal.log()。
*/

contract Foo {
    Bar bar;

    constructor(address _bar) {
        bar = Bar(_bar);
    }

    function callBar() public {
        bar.log();
    }
}

contract Bar {
    event Log(string message);

    function log() public {
        emit Log("Bar was called");
    }
}

// 这段代码隐藏在一个单独的文件中
contract Mal {
    event Log(string message);

    // function () external {
    //     emit Log("Mal was called");
    // }

    // 实际上，即使没有这个函数，我们也可以通过
    // fallback 执行同样的攻击
    function log() public {
        emit Log("Mal was called");
    }
}
```

## 预防措施

- 在构造函数中初始化一个新合约
- 把外部合约的地址设为 `public`，以便可以审核
  外部合约的代码

```solidity
Bar public bar;

constructor() public {
    bar = new Bar();
}
```

---
## 关注我们
[Yanbo的Twitter](https://x.com/Yanbo2004)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)


[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
