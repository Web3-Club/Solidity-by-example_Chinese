# 基本数据类型

对应英文原页：https://solidity-by-example.org/primitives

这里向你介绍 Solidity 中一些可用的基本数据类型（primitive data types）。

- `boolean`
- `uint256`
- `int256`
- `address`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

contract Primitives {
    bool public boo = true;

    /*
    uint 代表无符号整数（unsigned integer），即非负整数
    有多种大小可选
        uint8   范围从 0 到 2 ** 8 - 1
        uint16  范围从 0 到 2 ** 16 - 1
        ...
        uint256 范围从 0 到 2 ** 256 - 1
    */
    uint8 public u8 = 1;
    uint256 public u256 = 456;
    uint256 public u = 123; // uint 是 uint256 的别名

    /*
    int 类型允许负数。
    与 uint 类似，从 int8 到 int256 有多种范围可选

    int256 范围从 -2 ** 255 到 2 ** 255 - 1
    int128 范围从 -2 ** 127 到 2 ** 127 - 1
    */
    int8 public i8 = -1;
    int256 public i256 = 456;
    int256 public i = -123; // int 与 int256 相同

    // int 的最小值和最大值
    int256 public minInt = type(int256).min;
    int256 public maxInt = type(int256).max;

    address public addr = 0xCA35b7d915458EF540aDe6068dFe2F44E8fa733c;

    /*
    在 Solidity 中，byte 数据类型表示一串字节。
    Solidity 提供两种字节类型：

     - 固定大小的字节数组
     - 动态大小的字节数组。

     Solidity 中的 bytes 一词表示动态字节数组。
     它是 byte[] 的简写。
    */
    bytes1 a = 0xb5; //  [10110101]
    bytes1 b = 0x56; //  [01010110]

    // 默认值
    // 未赋值的变量有一个默认值
    bool public defaultBoo; // false
    uint256 public defaultUint; // 0
    int256 public defaultInt; // 0
    address public defaultAddr; // 0x0000000000000000000000000000000000000000
}
```

---
## 关注我们
[Yanbo的Twitter](https://x.com/Yanbo2004)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)


[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
