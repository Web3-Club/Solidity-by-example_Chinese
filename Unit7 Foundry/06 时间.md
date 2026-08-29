# 时间

对应英文原页：https://solidity-by-example.org/foundry/time

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

import "forge-std/Test.sol";

contract TimeTest is Test {
    // vm.warp - 将 block.timestamp 设置为未来时间戳
    // vm.roll - 设置 block.number
    // skip - 增加当前时间戳
    // rewind - 减少当前时间戳

    function test() public {
        console.log("timestamp", block.timestamp);
        console.log("block number", block.number);

        console.log("warp");
        vm.warp(block.timestamp + 10);
        console.log("timestamp", block.timestamp);

        console.log("skip");
        skip(10);
        console.log("timestamp", block.timestamp);

        console.log("roll");
        vm.roll(10);
        console.log("block number", block.number);

        console.log("rewind");
        rewind(10);
        console.log("timestamp", block.timestamp);
    }
}

```

---
## 关注我们
[Yanbo的Twitter](https://x.com/Yanbo2004)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)


[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
