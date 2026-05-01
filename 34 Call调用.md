# Call调用

`call` 是一种与其他合约交互的低级函数。

当仅通过调用 `fallback` 函数来发送以太币时，这是推荐的方法。但不推荐用它来调用已存在的函数。

## 不推荐使用低级 call 的几个原因

- 错误不会向上冒泡（revert 不会传播）
- 绕过了类型检查
- 跳过了函数存在性检查

```solidity
// SPDX-License-Identifier: MIT  // SPDX 许可证标识符：MIT 许可证
pragma solidity ^0.8.26;  // 版本指令，要求使用 Solidity 0.8.26 及以上版本

// Receiver 合约：被调用的目标合约
contract Receiver {
    // Received 事件：记录调用者和调用信息
    event Received(address caller, uint256 amount, string message);

    // fallback 函数：当调用了不存在的函数时触发
    fallback() external payable {
        emit Received(msg.sender, msg.value, "Fallback was called");  // 记录 fallback 被调用
    }

    // foo 函数：一个测试函数，接收字符串和数字参数
    function foo(string memory _message, uint256 _x)
        public
        payable
        returns (uint256)  // 返回 _x + 1
    {
        emit Received(msg.sender, msg.value, _message);  // 记录调用信息

        return _x + 1;  // 返回参数 _x 加 1
    }
}

// Caller 合约：使用 call 调用 Receiver 合约
contract Caller {
    // Response 事件：记录调用是否成功及返回数据
    event Response(bool success, bytes data);

    // 假设 Caller 合约没有 Receiver 合约的源码，
    // 但我们知道 Receiver 的地址和要调用的函数签名。
    // testCallFoo 函数：使用 call 调用 Receiver 的 foo 函数
    function testCallFoo(address payable _addr) public payable {  // 接收目标地址
        // 可以通过 call 发送以太币并指定自定义 gas 量
        // abi.encodeWithSignature 将函数签名和参数编码为 calldata
        (bool success, bytes memory data) = _addr.call{
            value: msg.value,  // 发送的以太币数量
            gas: 5000           // 指定的 gas 上限
        }(abi.encodeWithSignature("foo(string,uint256)", "call foo", 123));  // 编码函数签名和参数

        emit Response(success, data);  // 记录调用结果
    }

    // testCallDoesNotExist 函数：调用不存在的函数，将触发 fallback
    function testCallDoesNotExist(address payable _addr) public payable {  // 接收目标地址
        (bool success, bytes memory data) = _addr.call{value: msg.value}(
            abi.encodeWithSignature("doesNotExist()")  // 这个函数在 Receiver 中不存在
        );

        emit Response(success, data);  // 记录调用结果（success 仍为 true，因为 fallback 执行成功）
    }
}
```
