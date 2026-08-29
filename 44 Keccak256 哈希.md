# Keccak256 哈希

对应英文原页：https://solidity-by-example.org/hashing

`keccak256` 计算输入的 Keccak-256 哈希。

一些用例：

- 根据输入生成确定性的唯一 ID
- 提交-揭示（Commit-Reveal）方案
- 紧凑的密码学签名（对哈希签名，而不是对更大的输入签名）

Solidity 提供两种编码数据的方法：

- `abi.encode`：
  - 带填充（padding）地把数据编码成字节
  - 保留全部数据信息
  - 处理动态类型时更安全
  - 由于填充，输出更长
- `abi.encodePacked`：
  - 执行紧凑编码（packed encoding，压缩）
  - 输出比 `abi.encode` 更短
  - 更节省 gas
  - 对动态类型存在哈希碰撞（hash collision）风险（见 `collision` 函数）

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

contract HashFunction {
    function hash(string memory _text, uint256 _num, address _addr)
        public
        pure
        returns (bytes32)
    {
        return keccak256(abi.encodePacked(_text, _num, _addr));
    }

    // 哈希碰撞示例
    // 当你向 abi.encodePacked 传入多个动态数据类型时，可能发生哈希碰撞。
    // 这种情况下应改用 abi.encode。
    function collision(string memory _text, string memory _anotherText)
        public
        pure
        returns (bytes32)
    {
        // encodePacked(AAA, BBB) -> AAABBB
        // encodePacked(AA, ABBB) -> AAABBB
        return keccak256(abi.encodePacked(_text, _anotherText));
    }
}

contract GuessTheMagicWord {
    bytes32 public answer =
        0x60298f78cc0b47170ba79c10aa3851d7648bd96f2f8e46a19dbc777c36fb0c00;

    // 魔法词是 "Solidity"
    function guess(string memory _word) public view returns (bool) {
        return keccak256(abi.encodePacked(_word)) == answer;
    }
}
```

---
## 关注我们
[Yanbo的Twitter](https://x.com/Yanbo2004)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)


[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
