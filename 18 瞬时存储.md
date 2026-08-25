# 瞬时存储

对应英文原页：https://solidity-by-example.org/transient-storage

存储在瞬时存储（transient storage）中的数据会在交易结束后清除。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

// 请确保 EVM 版本和 VM 设置为 Cancun

// Storage - 数据存储在区块链上
// Memory - 数据在函数调用结束后清除
// Transient storage - 数据在交易结束后清除

interface ITest {
    function val() external view returns (uint256);
    function test() external;
}

// 用于测试 TestStorage 和 TestTransientStorage 的合约
// 展示普通 storage 与瞬时存储的区别
contract Callback {
    uint256 public val;

    fallback() external {
        val = ITest(msg.sender).val();
    }

    function test(address target) external {
        ITest(target).test();
    }
}

contract TestStorage {
    uint256 public val;

    function test() public {
        val = 123;
        bytes memory b = "";
        msg.sender.call(b);
    }
}

contract TestTransientStorage {
    bytes32 constant SLOT = 0;

    function test() public {
        assembly {
            tstore(SLOT, 321)
        }
        bytes memory b = "";
        msg.sender.call(b);
    }

    function val() public view returns (uint256 v) {
        assembly {
            v := tload(SLOT)
        }
    }
}

// 用于测试重入（re-entrancy）保护的合约
contract MaliciousCallback {
    uint256 public count = 0;

    // 尝试多次重入目标合约
    fallback() external {
        ITest(msg.sender).test();
    }

    // 用于发起重入攻击的测试函数
    function attack(address _target) external {
        // 第一次调用 test()
        ITest(_target).test();
    }
}

contract ReentrancyGuard {
    bool private locked;

    modifier lock() {
        require(!locked);
        locked = true;
        _;
        locked = false;
    }

    // 27587 gas
    function test() public lock {
        // 忽略 call 的错误
        bytes memory b = "";
        msg.sender.call(b);
    }
}

contract ReentrancyGuardTransient {
    bytes32 constant SLOT = 0;

    modifier lock() {
        assembly {
            if tload(SLOT) { revert(0, 0) }
            tstore(SLOT, 1)
        }
        _;
        assembly {
            tstore(SLOT, 0)
        }
    }

    // 4909 gas
    function test() external lock {
        // 忽略 call 的错误
        bytes memory b = "";
        msg.sender.call(b);
    }
}
```

---
## 关注我们
[Yanbo的Twitter](https://x.com/Yanbo2004)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)


[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
