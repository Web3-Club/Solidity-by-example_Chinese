# delegatecall() 函数

> 本文介绍了 Solidity 中的 `delegatecall()` 函数及其使用。

delegatecall 是类似于调用的低级函数。 

当合约A对合约B执行delegatecall时，执行B的代码 与合约 A 的存储，msg.sender 和 msg.value。

## 示例

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

// NOTE: Deploy this contract first
contract B {
    // NOTE: storage layout must be the same as contract A
    uint public num;
    address public sender;
    uint public value;

    function setVars(uint _num) public payable {
        num = _num;
        sender = msg.sender;
        value = msg.value;
    }
}

contract A {
    uint public num;
    address public sender;
    uint public value;

    function setVars(address _contract, uint _num) public payable {
        // A's storage is set, B is not modified.
        (bool success, bytes memory data) = _contract.delegatecall(
            abi.encodeWithSignature("setVars(uint256)", _num)
        );
    }
}

```

在上面的示例中，我们定义了两个智能合约 `A` 和 `B`。`A` 合约包含一个名为 `foo()` 的函数，该函数接受两个数字参数并返回它们的总和。`B` 合约包含一个名为 `bar()` 的函数，该函数向另一个合约地址发送一个 `delegatecall()`，并将 `foo()` 函数的输入参数 `_i` 和 `_j` 作为参数传递。`bar()` 函数将返回 `foo()` 函数返回值的总和。

请注意，我们使用了 `delegatecall()` 函数来实现这个过程，并使用 `abi.encodeWithSignature()` 函数对参数进行编码。

## 详解

### `delegatecall()` 函数概述

`delegatecall()` 是 Solidity 中的一个特殊函数，用于在当前合约的上下文中执行其他合约的代码。与 `call()` 不同，`delegatecall()` 将使用当前合约的存储器、状态变量和 ETH 余额。这意味着我们可以将另一个合约的代码视为当前合约的一部分，并且可以使用当前合约的所有资源。

### 函数编码

在使用 `delegatecall()` 函数调用其他合约中的函数时，我们需要对函数参数进行编码。Solidity 提供了多种编码函数，可以根据不同的参数类型来选择使用适当的函数。例如，我们可以使用 `abi.encodeWithSignature()` 函数对数字参数进行编码。

```solidity
bytes memory payload = abi.encodeWithSignature("foo(uint256,uint256)", _i, _j);
```

在上面的示例中，我们使用 `abi.encodeWithSignature()` 函数对 `_i` 和 `_j` 进行编码，并将编码后的字节数组存储在名为 `payload` 的变量中。

请注意，使用错误的编码方式可能会导致调用失败或产生错误的返回值。

### `require()` 函数

使用 `require()` 函数来检查条件是否满足，并在条件不满足时抛出异常。

```solidity
require(success);
```

在上面的示例中，我们使用 `require()` 函数来检查 `delegatecall()` 是否成功，并在不成功时抛出异常。

## 总结

通过使用 `delegatecall()` 函数和正确的函数参数编码方式，Solidity 可以方便地处理智能合约之间的代码共享。程序员可以使用这些工具来执行复杂的逻辑或转移场景，并确保正确地处理返回值和错误情况。通过使用 `require()` 函数，程序员可以轻松地检查条件是否满足，并在必要时抛出异常。