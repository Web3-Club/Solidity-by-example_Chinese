# call() 函数

> 本文介绍了 Solidity 中的 `call()` 函数及其使用。

call 是与其他合约交互的低级函数。

当您只是通过调用回退函数发送 Ether 时，这是推荐使用的方法。

但是，这不是调用现有函数的推荐方式。

不推荐低级调用的几个原因
- 还原不会冒泡
- 绕过类型检查
- 函数存在性检查被省略
## 示例

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

contract Receiver {
    event Received(address caller, uint amount, string message);

    fallback() external payable {
        emit Received(msg.sender, msg.value, "Fallback was called");
    }

    function foo(string memory _message, uint _x) public payable returns (uint) {
        emit Received(msg.sender, msg.value, _message);

        return _x + 1;
    }
}

contract Caller {
    event Response(bool success, bytes data);

    // Let's imagine that contract Caller does not have the source code for the
    // contract Receiver, but we do know the address of contract Receiver and the function to call.
    function testCallFoo(address payable _addr) public payable {
        // You can send ether and specify a custom gas amount
        (bool success, bytes memory data) = _addr.call{value: msg.value, gas: 5000}(
            abi.encodeWithSignature("foo(string,uint256)", "call foo", 123)
        );

        emit Response(success, data);
    }

    // Calling a function that does not exist triggers the fallback function.
    function testCallDoesNotExist(address payable _addr) public payable {
        (bool success, bytes memory data) = _addr.call{value: msg.value}(
            abi.encodeWithSignature("doesNotExist()")
        );

        emit Response(success, data);
    }
}

```

在上面的示例中，我们定义了两个智能合约 `Caller` 和 `Callee`。`Caller` 合约包含一个名为 `testCall()` 的函数，该函数向指定地址发送 1 wei 的以太币，并调用 `Callee` 合约中的 `foo()` 函数。当 `foo()` 函数被调用时，它将返回一个编码后的字符串和数字。

请注意，我们使用了 `call()` 函数来实现这个过程，并使用 `abi.encodeWithSignature()` 函数对参数进行编码。

这是一个测试从多重签名钱包发送交易的合同：

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

contract TestContract {
    uint public i;

    function callMe(uint j) public {
        i += j;
    }

    function getData() public pure returns (bytes memory) {
        return abi.encodeWithSignature("callMe(uint256)", 123);
    }
}

```

## 详解

### `call()` 函数概述

`call()` 是 Solidity 中的一个特殊函数，用于调用其他合约中的函数。在调用 `call()` 函数时，我们需要提供要调用函数的地址、以太币数量和参数。`call()` 函数将返回一个布尔值和一个 `bytes` 类型的数组，用于表示调用是否成功以及包含返回值的数据。

### 函数编码

在使用 `call()` 函数调用其他合约中的函数时，我们需要对函数参数进行编码。Solidity 提供了多种编码函数，可以根据不同的参数类型来选择使用适当的函数。例如，我们可以使用 `abi.encodeWithSignature()` 函数对字符串和数字参数进行编码。

```solidity
bytes memory data = abi.encodeWithSignature("foo(string,uint256)", "hello", 123);
```

在上面的示例中，我们使用 `abi.encodeWithSignature()` 函数对 `"hello"` 和 `123` 进行编码，并将编码后的字节数组存储在名为 `data` 的变量中。

请注意，使用错误的编码方式可能会导致调用失败或产生错误的返回值。

### `address payable` 类型

在 Solidity 中，可以使用 `address payable` 类型来表示一个既可以接收以太币又可以发送以太币的地址。

```solidity
address payable myAddress = payable(0x123);
```

在上面的示例中，我们将地址 `0x123` 转换为 `address payable` 类型，并将其存储在名为 `myAddress` 的变量中。

请注意，只有具有 `payable` 函数的地址才能被转换为 `address payable` 类型。

## 总结

通过使用 `call()` 函数和正确的函数参数编码方式，Solidity 可以方便地处理智能合约之间的函数调用。程序员可以使用这些工具来执行复杂的逻辑或转移场景，并确保正确地处理返回值和错误情况。通过使用 `address payable` 类型，程序员可以轻松地实现与其他合约交互并处理以太币转移。