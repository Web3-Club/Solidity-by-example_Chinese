# Keccak256哈希

`keccak256` 用于计算输入的 Keccak-256 哈希值。

一些用例包括：

- 从输入创建确定性的唯一 ID
- 提交-揭示（Commit-Reveal）方案
- 紧凑的加密签名（通过签名哈希值而不是较大的原始输入）

Solidity 提供了两种数据编码方法：

| 方法 | 特点 | 优势 | 劣势 |
|------|------|------|------|
| `abi.encode` | 带填充的编码，保留所有数据信息 | 处理动态类型时更安全 | 输出较长 |
| `abi.encodePacked` | 紧凑编码（压缩），Gas 更高效 | 输出更短，节省 Gas | 动态类型存在哈希碰撞风险 |

```solidity
// SPDX-License-Identifier: MIT  // SPDX 许可证标识符：MIT 许可证
pragma solidity ^0.8.26;  // 版本指令，要求使用 Solidity 0.8.26 及以上版本

// HashFunction 合约：演示 keccak256 哈希的用法及碰撞风险
contract HashFunction {  // 定义 HashFunction 合约
    // hash 函数：对输入进行 keccak256 哈希
    function hash(  // 接收字符串、uint 和地址三种类型参数
        string memory _text,  // 文本字符串
        uint256 _num,         // 数值
        address _addr         // 地址
    ) public pure returns (bytes32) {  // 返回 32 字节的哈希值
        // abi.encodePacked 执行紧凑编码后进行哈希
        // 注意：多个动态类型组合时存在哈希碰撞风险
        return keccak256(abi.encodePacked(_text, _num, _addr));  // 打包编码后计算哈希
    }

    // collision 函数：演示 abi.encodePacked 的哈希碰撞漏洞
    // 当你使用 abi.encodePacked 打包多个动态类型（如字符串）时，
    // 不同的输入组合可能产生相同的打包结果，从而导致相同的哈希值
    function collision(string memory _text, string memory _anotherText)  // 接收两个字符串
        public
        pure
        returns (bytes32)  // 返回哈希值
    {
        // encodePacked("AAA", "BBB") -> "AAABBB"
        // encodePacked("AA", "ABBB")  -> "AAABBB"
        // 以上两种不同的输入产生了相同的打包结果！
        // 因此它们的 keccak256 哈希值也完全相同，这就是哈希碰撞
        return keccak256(abi.encodePacked(_text, _anotherText));  // 存在碰撞风险的写法
    }

    // 修复方法：使用 abi.encode 替代 abi.encodePacked
    // abi.encode 会在每个参数之间添加 32 字节的填充，避免边界模糊
    // keccak256(abi.encode(_text, _anotherText)) 是更安全的写法
}

// GuessTheMagicWord 合约：演示 Commit-Reveal 模式
contract GuessTheMagicWord {  // 定义猜词合约
    // 预先存储正确答案的哈希值
    // 正确答案是 "Solidity"，但哈希值不会泄露原始信息
    bytes32 public answer =
        0x60298f78cc0b47170ba79c10aa3851d7648bd96f2f22e46facf0db0fa7c77189;  // keccak256(abi.encodePacked("Solidity"))

    // guess 函数：用户提交猜测，验证是否匹配
    function guess(string memory _word) public view returns (bool) {  // 接收用户猜的词
        // 对用户的输入进行哈希，与预设的 answer 进行比较
        return keccak256(abi.encodePacked(_word)) == answer;  // 匹配则返回 true
    }
}
```
