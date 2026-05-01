# payable 函数

> 本文介绍了 Solidity 中的 `payable` 函数修饰符，它允许合约接收以太币。

声明为 payable 的函数和地址可以将以太币接收到合约中。

## 示例

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

contract Payable {
    // Payable地址可以接收以太币
    address payable public owner;

    // Payable构造函数可以接收以太币
    constructor() payable {
        owner = payable(msg.sender);
    }

    // 存入以太币到智能合约的函数。
    // 与一些以太币一起调用此函数。
    // 智能合约的余额将自动更新。
    function deposit() public payable {}

    // 与一些以太币一起调用此函数。
    // 由于此函数不可支付，因此该函数将抛出错误。
    function notPayable() public {}

    // 从智能合约中提取所有以太币的函数。
    function withdraw() public {
        // 获取存储在此合约中的以太币数量
        uint amount = address(this).balance;

        // 将所有以太币发送给拥有者
        // 由于所有者的地址是可支付的，因此其可以接收以太币
        (bool success, ) = owner.call{value: amount}("");
        require(success, "Failed to send Ether");
    }

    // 将以太币从此合约转移到输入地址的函数
    function transfer(address payable _to, uint _amount) public {
        // 注意，“to”被声明为payable
        (bool success, ) = _to.call{value: _amount}("");
        require(success, "Failed to send Ether");
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