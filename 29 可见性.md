# 可见性

对应英文原页：https://solidity-by-example.org/visibility

函数和状态变量必须声明它们是否可被其他合约访问。

函数可以声明为

- `public` - 任何合约和账户都可以调用
- `private` - 仅能在定义该函数的合约内部调用
- `internal` - 仅能在继承了 `internal` 函数的合约内部调用
- `external` - 仅其他合约和账户可以调用

状态变量可以声明为 `public`、`private` 或 `internal`，但不能声明为 `external`。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

contract Base {
    // 私有函数只能被调用：
    // - 在本合约内部
    // 继承本合约的合约无法调用此函数。
    function privateFunc() private pure returns (string memory) {
        return "private function called";
    }

    function testPrivateFunc() public pure returns (string memory) {
        return privateFunc();
    }

    // 内部函数可以被调用：
    // - 在本合约内部
    // - 在继承本合约的合约内部
    function internalFunc() internal pure returns (string memory) {
        return "internal function called";
    }

    function testInternalFunc() public pure virtual returns (string memory) {
        return internalFunc();
    }

    // 公开函数可以被调用：
    // - 在本合约内部
    // - 在继承本合约的合约内部
    // - 由其他合约和账户
    function publicFunc() public pure returns (string memory) {
        return "public function called";
    }

    // 外部函数只能被调用：
    // - 由其他合约和账户
    function externalFunc() external pure returns (string memory) {
        return "external function called";
    }

    // 此函数无法编译，因为我们试图在这里调用
    // 一个外部函数。
    // function testExternalFunc() public pure returns (string memory) {
    //     return externalFunc();
    // }

    // 状态变量
    string private privateVar = "my private variable";
    string internal internalVar = "my internal variable";
    string public publicVar = "my public variable";
    // 状态变量不能是 external，因此这段代码无法编译。
    // string external externalVar = "my external variable";
}

contract Child is Base {
    // 继承的合约无法访问私有函数
    // 和状态变量。
    // function testPrivateFunc() public pure returns (string memory) {
    //     return privateFunc();
    // }

    // 内部函数可以在子合约内部被调用。
    function testInternalFunc() public pure override returns (string memory) {
        return internalFunc();
    }
}
```

---
## 关注我们
[Yanbo的Twitter](https://x.com/Yanbo2004)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)


[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
