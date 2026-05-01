# Unchecked数学运算

在 Solidity 0.8 及以上版本中，数字的溢出和下溢默认会抛出错误。可以通过使用 `unchecked` 来禁用此检查。

禁用溢出/下溢检查可以节省 Gas。

```solidity
// SPDX-License-Identifier: MIT  // SPDX 许可证标识符：MIT 许可证
pragma solidity ^0.8.26;  // 版本指令，要求使用 Solidity 0.8.26 及以上版本

// UncheckedMath 合约：演示使用 unchecked 绕过溢出检查以节省 Gas
contract UncheckedMath {  // 定义 UncheckedMath 合约
    // add 函数：使用 unchecked 进行加法运算
    function add(uint256 x, uint256 y) public pure returns (uint256) {
        // unchecked 块内的数学运算不会检查溢出
        unchecked {  // 禁用溢出检查
            return x + y;  // 约 22103 gas（比默认的 22291 gas 节省约 188 gas）
        }
        // 注意：如果溢出发生，结果会静默回绕（wrapping），不会 revert
    }

    // sub 函数：使用 unchecked 进行减法运算
    function sub(uint256 x, uint256 y) public pure returns (uint256) {
        unchecked {  // 禁用溢出/下溢检查
            return x - y;  // 约 22147 gas（比默认的 22329 gas 节省约 182 gas）
        }
        // 注意：如果 y > x，结果会下溢回绕为一个大数
    }

    // sumOfCubes 函数：在 unchecked 中进行多次数学运算
    // 当开发者确定不会发生溢出时，将复杂数学逻辑放入 unchecked 可以显著节省 Gas
    function sumOfCubes(uint256 x, uint256 y) public pure returns (uint256) {
        // 将复杂的数学逻辑包装在 unchecked 块中
        unchecked {  // 整个表达式在禁用溢出检查的情况下计算
            // 计算 x³ + y³
            uint256 x3 = x * x * x;  // x 的三次方（不检查溢出）
            uint256 y3 = y * y * y;  // y 的三次方（不检查溢出）
            return x3 + y3;  // 返回 x³ + y³（不检查溢出）
        }
    }

    // 风险提示：如果 unchecked 块内确实发生了溢出，结果会静默回绕，
    // 不会像正常 Solidity 代码那样抛出错误并回滚交易。
    // 因此，只有在开发者 100% 确定不会溢出时才应使用 unchecked。
}
```
