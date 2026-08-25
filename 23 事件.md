# 事件

对应英文原页：https://solidity-by-example.org/events

`Events`（事件）允许向以太坊区块链写入日志。事件的一些用例包括：

- 监听事件并更新用户界面
- 一种廉价的存储方式

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

contract Event {
    // 事件声明
    // 最多可以对 3 个参数建立索引。
    // 带索引的参数帮助你按该参数过滤日志
    event Log(address indexed sender, string message);
    event AnotherLog();

    function test() public {
        emit Log(msg.sender, "Hello World!");
        emit Log(msg.sender, "Hello EVM!");
        emit AnotherLog();
    }
}
```

---
## 关注我们
[Yanbo的Twitter](https://x.com/Yanbo2004)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)


[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
