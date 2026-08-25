# 发送以太币

对应英文原页：https://solidity-by-example.org/sending-ether

### 如何发送以太币？

你可以通过以下方式向其他合约发送以太币：

- `transfer`（2300 gas，会抛出错误）
- `send`（2300 gas，返回 bool）
- `call`（转发全部 gas 或指定 gas，返回 bool）

### 如何接收以太币？

接收以太币的合约必须至少具备以下函数之一：

- `receive() external payable`
- `fallback() external payable`

如果 `msg.data` 为空则调用 `receive()`，否则调用 `fallback()`。

### 应该使用哪种方法？

2019 年 12 月之后，推荐将 `call` 与重入保护（re-entrancy guard）结合使用。

通过以下方式防范重入（re-entrancy）：

- 在调用其他合约之前完成所有状态更改
- 使用重入保护修饰符

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

contract ReceiveEther {
    /*
    调用的是哪个函数，fallback() 还是 receive()？

           发送以太币
               |
         msg.data 为空？
              / \
            是    否
            /     \
    存在 receive()？  fallback()
         /   \
        是     否
        /      \
    receive()   fallback()
    */

    // 用于接收以太币的函数。msg.data 必须为空
    receive() external payable {}

    // 当 msg.data 不为空时会调用回退函数（fallback）
    fallback() external payable {}

    function getBalance() public view returns (uint256) {
        return address(this).balance;
    }
}

contract SendEther {
    function sendViaTransfer(address payable _to) public payable {
        // 此函数已不再被推荐用于发送以太币。
        _to.transfer(msg.value);
    }

    function sendViaSend(address payable _to) public payable {
        // send 返回一个表示成功或失败的布尔值。
        // 不推荐使用此函数发送以太币。
        bool sent = _to.send(msg.value);
        require(sent, "Failed to send Ether");
    }

    function sendViaCall(address payable _to) public payable {
        // call 返回一个表示成功或失败的布尔值。
        // 这是当前推荐使用的方法。
        (bool sent, bytes memory data) = _to.call{value: msg.value}("");
        require(sent, "Failed to send Ether");
    }
}
```

---
## 关注我们
[Yanbo的Twitter](https://x.com/Yanbo2004)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)


[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
