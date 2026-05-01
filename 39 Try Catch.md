# Try Catch

`try / catch` 只能捕获来自**外部函数调用**和**合约创建**的错误。

```solidity
// SPDX-License-Identifier: MIT  // SPDX 许可证标识符：MIT 许可证
pragma solidity ^0.8.26;  // 版本指令，要求使用 Solidity 0.8.26 及以上版本

// Foo 合约：用于 try / catch 示例的外部合约
contract Foo {
    address public owner;  // 公共变量：合约所有者地址

    // 构造函数：创建合约时设置所有者
    constructor(address _owner) {  // 接收一个地址作为所有者
        require(_owner != address(0), "invalid address");  // require 检查：所有者不能是零地址
        // assert 检查：所有者不能是特定地址（0x000...001）
        // 使用 assert 触发 Panic 错误，区别于 require 的 Error 错误
        assert(_owner != 0x0000000000000000000000000000000000000001);
        owner = _owner;  // 设置所有者
    }

    // myFunc 函数：一个测试函数
    function myFunc(uint256 x) public pure returns (string memory) {  // 接收一个 uint256 参数
        require(x != 0, "require failed");  // 如果 x 为 0，触发 require 错误
        return "my func was called";  // 成功时返回此字符串
    }
}

// Bar 合约：演示 try / catch 的用法
contract Bar {
    event Log(string message);     // 记录字符串消息的事件
    event LogBytes(bytes data);    // 记录原始字节数据的事件

    Foo public foo;  // Foo 合约实例

    constructor() {  // 构造函数
        // 在部署 Bar 合约时，同时创建一个 Foo 合约实例
        // 此 Foo 实例用于 try catch 外部调用示例
        foo = new Foo(msg.sender);  // 创建以当前部署者为所有者的 Foo 合约
    }

    // tryCatchExternalCall 函数：演示 try / catch 捕获外部调用错误
    // tryCatchExternalCall(0) => 输出 Log("external call failed")，因为 myFunc 中 require(x != 0) 失败
    // tryCatchExternalCall(1) => 输出 Log("my func was called")，调用成功
    function tryCatchExternalCall(uint256 _i) public {  // 接收一个测试参数
        try foo.myFunc(_i) returns (string memory result) {  // 尝试调用 foo.myFunc(_i)
            // 如果外部调用成功，执行此代码块
            emit Log(result);  // 记录返回的字符串
        } catch {  // 如果外部调用失败（任何类型的错误）
            // 执行此备用代码块
            emit Log("external call failed");  // 记录外部调用失败
        }
    }

    // tryCatchNewContract 函数：演示 try / catch 捕获合约创建错误
    // tryCatchNewContract(0x000...000) => Log("invalid address")，Require 失败
    // tryCatchNewContract(0x000...001) => LogBytes("")，Assert 失败（返回空数据）
    // tryCatchNewContract(0x000...002) => Log("Foo created")，创建成功
    function tryCatchNewContract(address _owner) public {  // 接收所有者地址参数
        try new Foo(_owner) returns (Foo foo) {  // 尝试创建新的 Foo 合约
            // 如果创建成功，可以在此使用 foo 变量
            emit Log("Foo created");  // 记录合约创建成功
        } catch Error(string memory reason) {  // 捕获 require() 和 revert() 产生的错误
            // 此类错误带有错误消息字符串
            emit Log(reason);  // 记录错误消息（如 "invalid address"）
        } catch (bytes memory reason) {  // 捕获 assert() 产生的 Panic 错误等其他低级错误
            // assert 失败时，错误数据通常为空
            emit LogBytes(reason);  // 记录原始错误数据
        }
    }
}
```
