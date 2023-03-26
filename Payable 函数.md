# payable 函数

> 本文介绍了 Solidity 中的 `payable` 函数修饰符，它允许合约接收以太币。

## 示例

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract ReceiveEther {
    // 当有人向这个合约发送以太币时，该函数会被调用
    receive() external payable {
    }
    
    // 向这个合约发送以太币
    function sendEther() external payable {
    }
    
    // 获取合约余额
    function getBalance() external view returns (uint) {
        return address(this).balance;
    }
}
```

在上面的示例中，我们定义了一个名为 `ReceiveEther` 的合约，并使用 `receive()` 函数作为其默认函数。如果有人向该合约发送以太币，则该函数将被调用。

我们还使用了 `payable` 修饰符来标识 `sendEther()` 函数，该函数允许向合约发送以太币。通过 `getBalance()` 函数，我们可以获取该合约的余额。

## 详解

### `receive()` 函数

当有人向合约发送以太币时，默认函数将被调用。默认函数是没有名称和参数的函数，如下所示：

```solidity
receive() external payable {
    // 处理接收到的以太币
}
```

请注意，如果合约没有定义 `receive()` 函数，并且有人向其发送以太币，则将会抛出异常并返还所有的以太币。

### `payable` 修饰符

使用 `payable` 修饰符修饰一个函数可以使该函数接收以太币。

```solidity
function myFunction() external payable {
    // 处理接收到的以太币
}
```

在上面的示例中，我们使用 `payable` 修饰符来标识 `myFunction()` 函数，该函数允许接收以太币。

请注意，只有具有 `payable` 修饰符的函数才能接收以太币。

### 合约余额

要获取合约的当前余额，可以使用 `address(this).balance`。

```solidity
function getBalance() external view returns (uint) {
    return address(this).balance;
}
```

在上面的示例中，我们定义了一个名为 `getBalance()` 的函数，该函数返回该合约的余额。请注意，我们使用 `view` 关键字来标识该函数不会修改合约状态。

## 总结

通过使用 `payable` 修饰符以及默认函数，Solidity 可以方便地处理合约接收以太币的情形。程序员可以使用这些工具来执行复杂的转账逻辑或支付场景，并在需要时获取当前合约的余额信息。