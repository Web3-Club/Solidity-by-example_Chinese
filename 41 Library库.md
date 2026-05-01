# Library库

库（Library）与合约类似，但不能声明任何状态变量，也不能发送以太币。

如果库中的所有函数都是 `internal` 的，则库会被嵌入到合约中。否则库必须被部署，并在合约部署前进行链接。

```solidity
// SPDX-License-Identifier: MIT  // SPDX 许可证标识符：MIT 许可证
pragma solidity ^0.8.26;  // 版本指令，要求使用 Solidity 0.8.26 及以上版本

// Math 库：提供数学工具函数
library Math {  // library 关键字定义库，类似合约但不能有状态变量
    // sqrt 函数：使用牛顿迭代法计算平方根
    function sqrt(uint256 y) internal pure returns (uint256 z) {  // internal 函数会被内嵌到调用合约中
        if (y > 3) {          // 如果 y 大于 3，使用牛顿迭代法
            z = y;            // 初始估计值设为 y
            uint256 x = y / 2 + 1;  // 初始猜测值
            while (x < z) {   // 当猜测值小于当前估计值时继续迭代
                z = x;        // 更新估计值
                x = (y / x + x) / 2;  // 牛顿迭代公式：x = (y/x + x) / 2
            }
        } else if (y != 0) {  // 如果 y 为 1、2 或 3
            z = 1;            // 平方根为 1（1²=1, 2²=4中的整数部分, 3²=9）
        }
        // 否则 z = 0（y 为 0 时返回默认值 0）
    }
}

// TestMath 合约：测试 Math 库的使用
contract TestMath {  // 定义测试合约
    // testSquareRoot 函数：调用 Math 库的 sqrt 函数
    function testSquareRoot(uint256 x) public pure returns (uint256) {
        return Math.sqrt(x);  // 通过库名直接调用函数
    }
}

// Array 库：提供数组操作函数
// 用于删除指定索引处的元素并重新整理数组，使元素之间没有空隙
library Array {  // 定义 Array 库
    // remove 函数：从数组中删除指定索引的元素（交换删除法）
    function remove(uint256[] storage arr, uint256 index) public {  // storage 引用，直接修改原数组
        // 将最后一个元素移到要删除的位置
        require(arr.length > 0, "Can't remove from empty array");  // 确保数组非空
        arr[index] = arr[arr.length - 1];  // 用最后一个元素覆盖要删除的位置
        arr.pop();  // 删除最后一个元素（因为已被移走）
    }
}

// TestArray 合约：测试 Array 库的使用
contract TestArray {  // 定义测试合约
    using Array for uint256[];  // using for 语法：将 Array 库附加到 uint256[] 类型

    uint256[] public arr;  // 声明一个公共 uint256 数组

    // testArrayRemove 函数：测试数组删除功能
    function testArrayRemove() public {
        // 循环添加 3 个元素
        for (uint256 i = 0; i < 3; i++) {  // i = 0, 1, 2
            arr.push(i);  // arr = [0, 1, 2]
        }

        // 调用 Array 库的 remove 函数删除索引 1 的元素（值为 1）
        arr.remove(1);  // arr 变为 [0, 2]（元素 1 被最后的 2 替换，然后 pop）

        // 验证结果
        assert(arr.length == 2);  // 数组长度应为 2
        assert(arr[0] == 0);      // 索引 0 应为 0
        assert(arr[1] == 2);      // 索引 1 应为 2
    }
}
```
