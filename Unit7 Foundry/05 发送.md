# 发送

对应英文原页：https://solidity-by-example.org/foundry/send

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

import "forge-std/Test.sol";

// deal 和 hoax 的示例
// deal(address, uint) - 设置地址的 ETH 余额
// deal(address, address, uint256) - 设置 ERC20 代币余额（适用于大多数代币）
// hoax(address, uint) - deal + prank

contract ERC20 {
    uint256 public totalSupply;
    mapping(address => uint256) public balanceOf;
}

contract SendTest is Test {
    ERC20 token = new ERC20();

    function testSendEth() public {
        // 设置 ETH 余额
        deal(address(1), 100);
        assertEq(address(1).balance, 100);

        // 设置 ERC20 余额
        deal(address(token), address(1), 10);
        assertEq(token.balanceOf(address(1)), 10);
    }
}

```

---
## 关注我们
[Yanbo的Twitter](https://x.com/Yanbo2004)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)


[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
