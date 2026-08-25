# Gas 优化

对应英文原页：https://solidity-by-example.org/gas-golf

一些节省 gas 的技巧。

- 用 `calldata` 替代 `memory`
- 把状态变量加载到内存
- 把 for 循环的 `i++` 换成 `++i`
- 缓存数组元素
- 短路求值（short circuit）

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

// gas golf
contract GasGolf {
    // 起始 - 50908 gas
    // 使用 calldata - 49163 gas
    // 把状态变量加载到内存 - 48952 gas
    // 短路求值 - 48634 gas
    // 循环递增优化 - 48244 gas
    // 缓存数组长度 - 48209 gas
    // 把数组元素加载到内存 - 48047 gas
    // 对 i 的溢出/下溢使用 unchecked - 47309 gas

    uint256 public total;

    // 起始版本 - 未做 gas 优化
    // function sumIfEvenAndLessThan99(uint[] memory nums) external {
    //     for (uint i = 0; i < nums.length; i += 1) {
    //         bool isEven = nums[i] % 2 == 0;
    //         bool isLessThan99 = nums[i] < 99;
    //         if (isEven && isLessThan99) {
    //             total += nums[i];
    //         }
    //     }
    // }

    // gas 优化后
    // [1, 2, 3, 4, 5, 100]
    function sumIfEvenAndLessThan99(uint256[] calldata nums) external {
        uint256 _total = total;
        uint256 len = nums.length;

        for (uint256 i = 0; i < len;) {
            uint256 num = nums[i];
            if (num % 2 == 0 && num < 99) {
                _total += num;
            }
            unchecked {
                ++i;
            }
        }

        total = _total;
    }
}
```

---
## 关注我们
[Yanbo的Twitter](https://twitter.com/YanboOfficial)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)

加入Web3Club 官方讨论群：YanboTravelAllWorld（微信号）

[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
