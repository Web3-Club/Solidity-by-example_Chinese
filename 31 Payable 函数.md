# payable 函数

> 本文介绍了 Solidity 中的 `payable` 函数修饰符，它允许合约接收以太币。

对应英文原页：https://solidity-by-example.org/payable

声明为 `payable` 的函数和地址可以将 `ether` 接收到合约中。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

contract Payable {
    // Payable 地址可以通过 transfer 或 send 发送以太币
    address payable public owner;

    // Payable 构造函数可以接收以太币
    constructor() payable {
        owner = payable(msg.sender);
    }

    // 向本合约存入以太币的函数。
    // 调用此函数时附带一些以太币。
    // 本合约的余额会自动更新。
    function deposit() public payable {}

    // 调用此函数时附带一些以太币。
    // 该函数会抛出错误，因为此函数不是 payable。
    function notPayable() public {}

    // 从本合约提取全部以太币的函数。
    function withdraw() public {
        // 获取存储在本合约中的以太币数量
        uint256 amount = address(this).balance;

        // 将全部以太币发送给 owner
        (bool success,) = owner.call{value: amount}("");
        require(success, "Failed to send Ether");
    }

    // 将以太币从本合约转到输入地址的函数
    function transfer(address payable _to, uint256 _amount) public {
        // 注意 "to" 被声明为 payable
        (bool success,) = _to.call{value: _amount}("");
        require(success, "Failed to send Ether");
    }
}
```

---
## 关注我们
[Yanbo的Twitter](https://x.com/Yanbo2004)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)


[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
