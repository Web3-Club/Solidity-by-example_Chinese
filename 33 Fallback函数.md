# Fallback函数

`fallback` 是一个特殊函数，在以下情况下执行：

- 调用了不存在的函数，或
- 直接向合约发送以太币，但 `receive()` 不存在或 `msg.data` 不为空

为了更好地理解 Solidity 在什么条件下调用 `receive` 或 `fallback` 函数，请参考以下流程图：

```
                 发送以太币
                      |
            msg.data 是否为空？
                /           \
              是             否
               |              |
      receive() 存在？    fallback()
          /        \
        是          否
         |           |
    receive()    fallback()
```

当通过 `transfer` 或 `send` 调用时，`fallback` 只能使用 2300 gas。

```solidity
// SPDX-License-Identifier: MIT  // SPDX 许可证标识符：MIT 许可证
pragma solidity ^0.8.26;  // 版本指令，要求使用 Solidity 0.8.26 及以上版本

// Fallback 合约：演示 fallback 和 receive 函数的行为
contract Fallback {
    // Log 事件：记录被调用的函数名和剩余 gas 量
    event Log(string func, uint256 gas);  // func 为函数名，gas 为剩余 gas 量

    // fallback 函数必须声明为 external
    // 当调用了不存在的函数，或 msg.data 不为空时触发
    fallback() external payable {
        // send / transfer 会将 2300 gas 转发到此 fallback 函数
        // call 会转发所有剩余的 gas
        emit Log("fallback", gasleft());  // 记录 "fallback" 和当前剩余 gas 量
    }

    // receive 是 fallback 的变体，当 msg.data 为空时触发
    receive() external payable {
        emit Log("receive", gasleft());  // 记录 "receive" 和当前剩余 gas 量
    }

    // getBalance 辅助函数：查询本合约的余额
    function getBalance() public view returns (uint256) {
        return address(this).balance;  // 返回本合约地址的以太币余额
    }
}

// SendToFallback 合约：演示向 Fallback 合约发送以太币时 fallback/receive 的行为
contract SendToFallback {
    // transferToFallback 函数：使用 transfer 方法发送以太币
    function transferToFallback(address payable _to) public payable {  // 接收目标地址
        _to.transfer(msg.value);  // transfer 仅转发 2300 gas
    }

    // callFallback 函数：使用 call 方法发送以太币
    function callFallback(address payable _to) public payable {  // 接收目标地址
        (bool sent,) = _to.call{value: msg.value}("");  // call 转发所有剩余 gas
        require(sent, "Failed to send Ether");  // 如果发送失败则回滚
    }
}
```

`fallback` 可以选择性地接收 `bytes` 类型的输入和输出：

```solidity
// SPDX-License-Identifier: MIT  // SPDX 许可证标识符：MIT 许可证
pragma solidity ^0.8.26;  // 版本指令，要求使用 Solidity 0.8.26 及以上版本

// FallbackInputOutput 合约：演示带输入输出的 fallback 函数
// 调用链：TestFallbackInputOutput -> FallbackInputOutput -> Counter
contract FallbackInputOutput {
    address immutable target;  // 不可变的目标合约地址

    // 构造函数：设置代理目标地址
    constructor(address _target) {
        target = _target;  // 初始化目标地址，部署后不可更改
    }

    // fallback 函数：接收 calldata 并转发给目标合约
    fallback(bytes calldata data) external payable returns (bytes memory) {  // 接收 calldata，返回 bytes
        (bool ok, bytes memory res) = target.call{value: msg.value}(data);  // 将调用转发到目标合约
        require(ok, "call failed");  // 如果调用失败则回滚
        return res;  // 返回目标合约的返回值
    }
}

// Counter 合约：一个简单的计数器合约，作为代理的目标合约
contract Counter {
    uint256 public count;  // 公共计数器变量

    // get 函数：获取当前计数值
    function get() external view returns (uint256) {
        return count;  // 返回当前计数
    }

    // inc 函数：计数器加一并返回新值
    function inc() external returns (uint256) {
        count += 1;  // 计数器自增 1
        return count;  // 返回更新后的计数值
    }
}

// TestFallbackInputOutput 合约：测试带输入输出 fallback 的调用
contract TestFallbackInputOutput {
    event Log(bytes res);  // Log 事件：记录返回数据

    // test 函数：向 fallback 合约发送调用
    function test(address _fallback, bytes calldata data) external {  // 接收 fallback 合约地址和调用数据
        (bool ok, bytes memory res) = _fallback.call(data);  // 向 fallback 合约发送调用
        require(ok, "call failed");  // 如果调用失败则回滚
        emit Log(res);  // 记录返回数据
    }

    // getTestData 函数：获取用于测试的编码数据
    function getTestData() external pure returns (bytes memory, bytes memory) {
        return
            (abi.encodeCall(Counter.get, ()), abi.encodeCall(Counter.inc, ()));  // 返回 Counter.get 和 Counter.inc 的 ABI 编码调用数据
    }
}
```
