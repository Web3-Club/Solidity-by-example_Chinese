# Gas优化技巧

一些节省 Gas 的技巧：

- 使用 `calldata` 替代 `memory`
- 将状态变量加载到内存中缓存
- 将 `i++` 替换为 `++i`
- 缓存数组元素
- 短路求值（Short Circuit）

```solidity
// SPDX-License-Identifier: MIT  // SPDX 许可证标识符：MIT 许可证
pragma solidity ^0.8.26;  // 版本指令，要求使用 Solidity 0.8.26 及以上版本

// GasGolf 合约：演示 Gas 优化技巧
contract GasGolf {  // 定义 GasGolf 合约
    uint256 public total;  // 总计数器（状态变量，存储在 storage 中）

    // 未优化版本（基准线：约 50908 gas）
    // sumIfEvenAndLessThan99 函数：对数组中满足条件的元素求和
    // 条件：偶数（num % 2 == 0）且小于 99
    function sumIfEvenAndLessThan99(uint256[] memory nums) external {  // 接收数组（memory）
        for (uint256 i = 0; i < nums.length; i++) {  // 遍历数组
            bool isEven = nums[i] % 2 == 0;           // 检查是否为偶数
            bool isLessThan99 = nums[i] < 99;          // 检查是否小于 99
            if (isEven && isLessThan99) {              // 同时满足两个条件
                total += nums[i];                       // 累加到 total（SSTORE 操作）
            }
        }
    }

    // 优化后的版本（约 47309 gas，节省约 7%）
    function sumIfEvenAndLessThan99Optimized(uint256[] calldata nums) external {  // 使用 calldata 替代 memory
        // 优化 1：将状态变量加载到内存中缓存，避免在循环中反复 SLOAD
        uint256 _total = total;  // 读取一次 storage 到内存，后续使用内存变量

        // 优化 2：缓存数组长度，避免每次循环都读取
        uint256 len = nums.length;  // 长度只读取一次

        // 优化 3：使用 ++i 替代 i++，并在 unchecked 中跳过溢出检查
        for (uint256 i = 0; i < len;) {  // 循环变量 i
            // 优化 4：将数组元素缓存到内存变量中
            uint256 num = nums[i];  // 避免重复从 calldata 读取

            // 优化 5：短路求值 —— 第一个条件为假时，第二个条件不计算
            // num % 2 == 0 为假时，直接跳过 && 后面的计算
            if (num % 2 == 0 && num < 99) {  // 短路求值节省不必要的比较
                _total += num;  // 累加到内存变量（不是 storage）
            }

            // 优化 3（续）：在 unchecked 块中自增，跳过溢出检查
            unchecked {
                ++i;  // i 不会溢出，跳过 SafeMath 检查以节省 gas
            }
        }

        // 循环结束后一次性写回 storage（仅一次 SSTORE）
        total = _total;  // 将最终结果写入状态变量
    }
}
```
