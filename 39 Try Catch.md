# Try / Catch

对应英文原页：https://solidity-by-example.org/try-catch

`try / catch` 只能捕获来自**外部函数调用**和**合约创建**的错误。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

// Foo 合约：用于 try / catch 示例的外部合约
contract Foo {
    address public owner;

    // 构造函数：创建合约时设置所有者
    constructor(address _owner) {
        // require 检查：所有者不能是零地址
        require(_owner != address(0), "invalid address");
        // assert 检查：所有者不能是特定地址（0x000...001）
        // 使用 assert 触发 Panic 错误，区别于 require 的 Error 错误
        assert(_owner != 0x0000000000000000000000000000000000000001);
        owner = _owner;
    }

    // myFunc 函数：一个测试函数
    function myFunc(uint256 x) public pure returns (string memory) {
        require(x != 0, "require failed");
        return "my func was called";
    }
}

// Bar 合约：演示 try / catch 的用法
contract Bar {
    event Log(string message);
    event LogBytes(bytes data);

    Foo public foo;

    constructor() {
        // 在部署 Bar 合约时，同时创建一个 Foo 合约实例
        // 此 Foo 实例用于 try catch 外部调用示例
        foo = new Foo(msg.sender);
    }

    // tryCatchExternalCall 函数：演示 try / catch 捕获外部调用错误
    // tryCatchExternalCall(0) => Log("external call failed")，因为 myFunc 中 require(x != 0) 失败
    // tryCatchExternalCall(1) => Log("my func was called")，调用成功
    function tryCatchExternalCall(uint256 _i) public {
        try foo.myFunc(_i) returns (string memory result) {
            // 如果外部调用成功，执行此代码块
            emit Log(result);
        } catch {
            // 如果外部调用失败（任何类型的错误），执行此备用代码块
            emit Log("external call failed");
        }
    }

    // tryCatchNewContract 函数：演示 try / catch 捕获合约创建错误
    // tryCatchNewContract(0x0000000000000000000000000000000000000000) => Log("invalid address")，Require 失败
    // tryCatchNewContract(0x0000000000000000000000000000000000000001) => LogBytes("")，Assert 失败（返回空数据）
    // tryCatchNewContract(0x0000000000000000000000000000000000000002) => Log("Foo created")，创建成功
    function tryCatchNewContract(address _owner) public {
        try new Foo(_owner) returns (Foo foo) {
            // 如果创建成功，可以在此使用 foo 变量
            emit Log("Foo created");
        } catch Error(string memory reason) {
            // 捕获 require() 和 revert() 产生的错误
            // 此类错误带有错误消息字符串
            emit Log(reason);
        } catch (bytes memory reason) {
            // 捕获 assert() 产生的 Panic 错误等其他低级错误
            // assert 失败时，错误数据通常为空
            emit LogBytes(reason);
        }
    }
}
```

---
## 关注我们
[Yanbo的Twitter](https://x.com/Yanbo2004)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)


[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
