# Chronicle 价格预言机

通过 Chronicle（原 Chronicle Protocol）去中心化预言机获取链上资产价格。Chronicle 是 MakerDAO 使用的预言机，具有高安全性和去中心化特性。

```solidity
// SPDX-License-Identifier: MIT  // SPDX 许可证标识符：MIT 许可证
pragma solidity ^0.8.16;  // 版本指令，要求使用 Solidity 0.8.16 及以上版本

// OracleReader 合约：从 Chronicle 预言机读取数据
// 完整仓库见 https://github.com/chronicleprotocol/OracleReader-Example
// 此合约地址硬编码为 Sepolia 测试网，其他网络请查阅 https://chroniclelabs.org/dashboard/oracles
contract OracleReader {  // 定义 OracleReader 合约
    // Chronicle ETH/USD 预言机地址（Sepolia 测试网）
    IChronicle public chronicle =
        IChronicle(address(0xdd6D76262Fd7BdDe428dcfCd94386EbAe0151603));

    // SelfKisser 地址：用于授权本合约访问 Chronicle 预言机
    // SelfKisser_1 地址（Sepolia 测试网）
    ISelfKisser public selfKisser =
        ISelfKisser(address(0x0Dcc19657007713483A5cA76e6A7bbe5f56EA37d));

    constructor() {
        // 将 address(this) 添加到 Chronicle 预言机的白名单中
        // 此操作使得本合约可以读取 Chronicle 预言机
        selfKisser.selfKiss(address(chronicle));  // 授权访问预言机
    }

    // read 函数：从 Chronicle 预言机读取最新数据
    function read() external view returns (uint256 val, uint256 age) {
        (val, age) = chronicle.readWithAge();  // 读取当前值和更新时间戳
    }
}

// IChronicle 接口：Chronicle 预言机标准接口
// 参考: https://github.com/chronicleprotocol/chronicle-std
interface IChronicle {  // 定义 IChronicle 接口
    // read 函数：返回预言机当前值（18 位小数）
    function read() external view returns (uint256 value);

    // readWithAge 函数：返回当前值及其更新时间戳
    function readWithAge() external view returns (uint256 value, uint256 age);
}

// ISelfKisser 接口：用于白名单授权
// 参考: https://github.com/chronicleprotocol/self-kisser
interface ISelfKisser {  // 定义 ISelfKisser 接口
    // selfKiss 函数：将调用者添加到指定预言机的白名单
    function selfKiss(address oracle) external;
}
```

## 与 Chainlink 的区别

| 特性 | Chronicle | Chainlink |
|------|-----------|-----------|
| 安全性模型 | 高去中心化验证者网络 | 去中心化预言机网络 |
| 授权机制 | 需要 SelfKisser 白名单授权 | 无需授权即可读取 |
| 主要用户 | MakerDAO 生态 | 通用 DeFi |
| 价格精度 | 18 位小数 | 因喂价对而异（如 8 位） |
