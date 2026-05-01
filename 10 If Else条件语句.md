# If Else条件语句

Solidity 支持条件语句 `if`、`else if` 和 `else`。

```solidity
// SPDX-License-Identifier: MIT  // SPDX 许可证标识符：MIT 许可证
pragma solidity ^0.8.26;  // 版本指令，要求使用 Solidity 0.8.26 及以上版本

contract IfElse {  // 定义名为 IfElse 的智能合约
    // foo 函数：演示 if / else if / else 多分支条件语句的用法
    function foo(uint256 x) public pure returns (uint256) {  // 接收一个 uint256 类型参数 x，返回一个 uint256 类型值
        if (x < 10) {  // 如果 x 小于 10
            return 0;  // 返回 0
        } else if (x < 20) {  // 否则如果 x 小于 20（即 10 <= x < 20）
            return 1;  // 返回 1
        } else {  // 否则（即 x >= 20）
            return 2;  // 返回 2
        }
    }

    // ternary 函数：演示三元运算符（条件表达式）的用法
    function ternary(uint256 _x) public pure returns (uint256) {  // 接收一个 uint256 类型参数 _x，返回一个 uint256 类型值
        // 以下是使用 if / else 语句的等价写法（已注释）：
        // if (_x < 10) {
        //     return 1;
        // }
        // return 2;

        // 三元运算符是 if / else 条件语句的简写形式
        // "?" 操作符被称为三元运算符
        // 语法格式：条件 ? 为真时的值 : 为假时的值
        return _x < 10 ? 1 : 2;  // 如果 _x < 10 成立（为真），返回 1；否则返回 2
    }
}
```
