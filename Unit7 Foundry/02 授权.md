# 授权

对应英文原页：https://solidity-by-example.org/foundry/auth

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

import {Test, console2, stdError} from "forge-std/Test.sol";

contract Auth {
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    function setOwner(address _owner) external {
        require(msg.sender == owner, "not authorized");
        owner = _owner;
    }
}

contract AuthTest is Test {
    Auth private auth;

    function setUp() public {
        // owner = 本合约
        auth = new Auth();
    }

    function testSetOwner() public {
        auth.setOwner(address(1));
        assertEq(auth.owner(), address(1));
    }

    function testFailNotOwner() public {
        // 下一次调用将由 address(1) 发起
        vm.prank(address(1));
        auth.setOwner(address(1));

        vm.startPrank(address(1));
        // 直到 stopPrank 之前的所有调用都由 address(1) 发起
        auth.setOwner(address(1));
        auth.setOwner(address(1));
        auth.setOwner(address(1));
        vm.stopPrank();
    }
}

```

---
## 关注我们
[Yanbo的Twitter](https://x.com/Yanbo2004)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)


[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
