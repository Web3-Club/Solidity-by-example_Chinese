# 最小代理合约

对应英文原页：https://solidity-by-example.org/app/minimal-proxy

如果你有一份会多次部署的合约，可以用最小代理合约（minimal proxy）以更低成本部署它们。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

// 原始代码
// https://github.com/optionality/clone-factory/blob/master/contracts/CloneFactory.sol

contract MinimalProxy {
    function clone(address target) external returns (address result) {
        // 把地址转换成 20 字节
        bytes20 targetBytes = bytes20(target);

        // 实际代码 //
        // 3d602d80600a3d3981f3363d3d373d3d3d363d73bebebebebebebebebebebebebebebebebebebebe5af43d82803e903d91602b57fd5bf3

        // 创建代码 //
        // 把运行时代码复制到内存并返回
        // 3d602d80600a3d3981f3

        // 运行时代码 //
        // 对地址执行 delegatecall 的代码
        // 363d3d373d3d3d363d73 address 5af43d82803e903d91602b57fd5bf3

        assembly {
            /*
            读取从 0x40 中保存的指针开始的 32 字节内存

            在 Solidity 中，内存的 0x40 槽比较特殊：它保存“空闲内存指针”，
            指向当前已分配内存的末尾。
            */
            let clone := mload(0x40)
            // 从 "clone" 开始向内存写入 32 字节
            mstore(
                clone,
                0x3d602d80600a3d3981f3363d3d373d3d3d363d73000000000000000000000000
            )

            /*
              |              20 bytes                |
            0x3d602d80600a3d3981f3363d3d373d3d3d363d73000000000000000000000000
                                                      ^
                                                      pointer
            */
            // 从 "clone" + 20 字节处开始向内存写入 32 字节
            // 0x14 = 20
            mstore(add(clone, 0x14), targetBytes)

            /*
              |               20 bytes               |                 20 bytes              |
            0x3d602d80600a3d3981f3363d3d373d3d3d363d73bebebebebebebebebebebebebebebebebebebebe
                                                                                              ^
                                                                                              pointer
            */
            // 从 "clone" + 40 字节处开始向内存写入 32 字节
            // 0x28 = 40
            mstore(
                add(clone, 0x28),
                0x5af43d82803e903d91602b57fd5bf30000000000000000000000000000000000
            )

            /*
              |               20 bytes               |                 20 bytes              |           15 bytes          |
            0x3d602d80600a3d3981f3363d3d373d3d3d363d73bebebebebebebebebebebebebebebebebebebebe5af43d82803e903d91602b57fd5bf3
            */
            // 创建新合约
            // 发送 0 Ether
            // 代码从 "clone" 中保存的指针开始
            // 代码大小 0x37（55 字节）
            result := create(0, clone, 0x37)
        }
    }
}
```

---
## 关注我们
[Yanbo的Twitter](https://x.com/Yanbo2004)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)


[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
