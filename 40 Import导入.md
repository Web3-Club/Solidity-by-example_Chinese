# Import导入

在 Solidity 中，你可以导入本地和外部文件。

## 本地导入

以下是我们的文件夹结构：

```
├── Import.sol
└── Foo.sol
```

Foo.sol —— 被导入的源文件：

```solidity
// SPDX-License-Identifier: MIT  // SPDX 许可证标识符：MIT 许可证
pragma solidity ^0.8.26;  // 版本指令，要求使用 Solidity 0.8.26 及以上版本

// Point 结构体：定义一个二维坐标点
struct Point {  // struct 关键字定义结构体类型
    uint256 x;  // x 坐标
    uint256 y;  // y 坐标
}

// Unauthorized 错误：自定义错误类型，携带调用者地址
error Unauthorized(address caller);  // error 关键字定义自定义错误

// add 函数：文件级别的自由函数（不属于任何合约）
function add(uint256 x, uint256 y) pure returns (uint256) {  // pure 表示不读取也不修改状态
    return x + y;  // 返回两数之和
}

// Foo 合约：被导入的合约
contract Foo {  // 定义名为 Foo 的智能合约
    string public name = "Foo";  // 公共字符串变量，默认值为 "Foo"
}
```

Import.sol —— 导入 Foo.sol 的文件：

```solidity
// SPDX-License-Identifier: MIT  // SPDX 许可证标识符：MIT 许可证
pragma solidity ^0.8.26;  // 版本指令，要求使用 Solidity 0.8.26 及以上版本

// 从当前目录导入 Foo.sol
import "./Foo.sol";  // 导入同一目录下的 Foo.sol 文件

// 命名导入：使用 {symbol1 as alias, symbol2} 语法
// 导入 Unauthorized 错误、add 函数（别名为 func）和 Point 结构体
import {Unauthorized, add as func, Point} from "./Foo.sol";  // 选择性导入并重命名

// Import 合约：使用导入的内容
contract Import {  // 定义名为 Import 的智能合约
    // 初始化 Foo.sol 中定义的合约
    Foo public foo = new Foo();  // 创建 Foo 合约的新实例

    // getFooName 函数：通过调用 Foo 合约的 name() 来测试导入是否成功
    function getFooName() public view returns (string memory) {  // 返回 Foo 合约的 name
        return foo.name();  // 调用 foo 实例的 name 函数
    }
}
```

## 外部导入

你也可以通过简单地复制 URL 从 [GitHub](https://github.com) 导入：

```solidity
// 基本格式：从 GitHub 仓库导入
// https://github.com/owner/repo/blob/branch/path/to/Contract.sol
import "https://github.com/owner/repo/blob/branch/path/to/Contract.sol";

// 示例：从 OpenZeppelin 的 openzeppelin-contracts 仓库导入 ECDSA.sol
// 仓库地址：https://github.com/OpenZeppelin/openzeppelin-contracts
// 分支：release-v4.5
import "https://github.com/OpenZeppelin/openzeppelin-contracts/blob/release-v4.5/contracts/utils/cryptography/ECDSA.sol";
```
