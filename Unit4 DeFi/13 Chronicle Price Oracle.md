# Chronicle 价格预言机

对应英文原页：https://solidity-by-example.org/defi/chronicle-price-oracle

### ETH / USD 价格预言机

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

/**
 * @title OracleReader
 * @notice 从 Chronicle 预言机读取数据的简单合约
 * @dev 完整仓库见 https://github.com/chronicleprotocol/OracleReader-Example。
 * @dev 本合约中的地址针对 Sepolia 测试网硬编码。
 * 其他受支持网络请查看 https://chroniclelabs.org/dashboard/oracles。
 */
contract OracleReader {
    /**
    * @notice 要读取的 Chronicle 预言机。
    * Chronicle_ETH_USD_3:0xdd6D76262Fd7BdDe428dcfCd94386EbAe0151603
    * Network: Sepolia
    */

    IChronicle public chronicle = IChronicle(address(0xdd6D76262Fd7BdDe428dcfCd94386EbAe0151603));

    /** 
    * @notice 为 Chronicle 预言机授予访问权限的 SelfKisser。
    * SelfKisser_1:0x0Dcc19657007713483A5cA76e6A7bbe5f56EA37d
    * Network: Sepolia
    * 不同测试网上 SelfKisser 地址的完整列表请见
    * https://docs.chroniclelabs.org/Developers/tutorials/Remix
    */
    ISelfKisser public selfKisser = ISelfKisser(address(0x0Dcc19657007713483A5cA76e6A7bbe5f56EA37d));

    constructor() {
        // 注意：将 address(this) 加入 Chronicle 预言机白名单。
        // 这样本合约才能从 Chronicle 预言机读取数据。
        selfKisser.selfKiss(address(chronicle));
    }

    /** 
    * @notice 从 Chronicle 预言机读取最新数据的函数。
    * @return val 预言机返回的当前值。
    * @return age 预言机上次更新的时间戳。
    */
    function read() external view returns (uint256 val, uint256 age) {
        (val, age) = chronicle.readWithAge();
    }
}

// 复制自 [chronicle-std](https://github.com/chronicleprotocol/chronicle-std/blob/main/src/IChronicle.sol)。
interface IChronicle {
    /** 
    * @notice 返回预言机的当前值。
    * @dev 若未设置值则回退。
    * @return value 预言机的当前值。
    */
    function read() external view returns (uint256 value);

    /** 
    * @notice 返回预言机的当前值及其年龄。
    * @dev 若未设置值则回退。
    * @return value 预言机的当前值，使用 18 位小数。
    * @return age 该值的年龄，为 Unix 时间戳。
    * */
    function readWithAge() external view returns (uint256 value, uint256 age);
}

// 复制自 [self-kisser](https://github.com/chronicleprotocol/self-kisser/blob/main/src/ISelfKisser.sol)。
interface ISelfKisser {
    /// @notice 将调用者 kiss 到该地址上。
    function selfKiss(address oracle) external;
}
```

---
## 关注我们
[Yanbo的Twitter](https://x.com/Yanbo2004)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)


[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
