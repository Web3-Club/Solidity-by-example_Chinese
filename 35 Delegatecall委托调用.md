# Delegatecall委托调用

`delegatecall` 是一种类似于 `call` 的低级函数。

当合约 `A` 对合约 `B` 执行 `delegatecall` 时，`B` 的代码会在 **`A` 的存储上下文**中执行，同时保留 `A` 的 `msg.sender` 和 `msg.value`。

这意味着 `delegatecall` 执行的是目标合约的**逻辑**，但读写的是调用合约的**存储**。

```solidity
// SPDX-License-Identifier: MIT  // SPDX 许可证标识符：MIT 许可证
pragma solidity ^0.8.26;  // 版本指令，要求使用 Solidity 0.8.26 及以上版本

// 注意：请先部署合约 B
// B 合约：被委托调用的合约（逻辑合约）
contract B {
    // 注意：存储布局必须与合约 A 相同
    uint256 public num;     // 插槽 0：数值变量
    address public sender;  // 插槽 1：发送者地址
    uint256 public value;   // 插槽 2：发送的以太币数量

    // setVars 函数：设置三个状态变量的值
    function setVars(uint256 _num) public payable {  // 接收数值参数，可附带以太币
        num = _num;          // 设置 num
        sender = msg.sender; // 设置 sender 为当前调用者
        value = msg.value;   // 设置 value 为附带的以太币数量
    }
}

// A 合约：调用 delegatecall 的合约（代理合约）
contract A {
    // 存储布局必须与合约 B 完全一致
    uint256 public num;     // 插槽 0：数值变量（与 B 对应）
    address public sender;  // 插槽 1：发送者地址（与 B 对应）
    uint256 public value;   // 插槽 2：发送的以太币数量（与 B 对应）

    // DelegateResponse 事件：记录 delegatecall 的结果
    event DelegateResponse(bool success, bytes data);
    // CallResponse 事件：记录普通 call 的结果
    event CallResponse(bool success, bytes data);

    // setVarsDelegateCall 函数：使用 delegatecall 调用合约 B 的 setVars
    // delegatecall 会修改 A 的存储，B 的存储不受影响
    function setVarsDelegateCall(address _contract, uint256 _num)
        public
        payable
    {
        // A 的存储被修改；B 的存储保持不变
        // delegatecall 使用 A 的存储上下文执行 B 的代码
        (bool success, bytes memory data) = _contract.delegatecall(
            abi.encodeWithSignature("setVars(uint256)", _num)  // 编码函数签名和参数
        );

        emit DelegateResponse(success, data);  // 记录 delegatecall 结果
    }

    // setVarsCall 函数：使用普通 call 调用合约 B 的 setVars（用于对比）
    // 普通 call 会修改 B 的存储，A 的存储不受影响
    function setVarsCall(address _contract, uint256 _num) public payable {
        // B 的存储被修改；A 的存储保持不变
        (bool success, bytes memory data) = _contract.call{value: msg.value}(
            abi.encodeWithSignature("setVars(uint256)", _num)  // 编码函数签名和参数
        );

        emit CallResponse(success, data);  // 记录 call 结果
    }
}
```
