# View 和 Pure 函数
## 概览
在 Solidity 0.5.0 以下版本中，任何内部函数（不包括 fallback 和 receive 函数）都可以声明为 constant。在 Solidity 0.5.0 及更高版本中，constant 被拆分成 view 和 pure。


### View 函数

View 函数指的是不修改合约状态的函数。如果调用 view 函数，则不会向区块链提交交易。我们可以将 view 视为函数的一个后缀，例如 getBalance 可以被重写为 getBalance view。对于其他合约调用 view 函数，也不会消耗任何 gas。

例子：

```
pragma solidity ^0.5.0;

contract Balance {
    mapping(address => uint) public balances;

    function getBalance() public view returns (uint) {
 return balances[msg.sender];
    }
}
```

在上面的合约中，`getBalance` 函数返回调用者的余额。因为这是一个 view 函数，所以在调用函数之后并不会修改合约的状态。因此，调用不会消耗任何 gas。

### Pure 函数

Pure 函数指的是没有在合约存储中读取或写入任何数据，并且不会改变合约状态，也不会读取区块链的任何数据。如果调用 pure 函数，则不会向区块链提交交易。我们可以将 pure 视为函数的另一个后缀，例如 add 可以被重写为 add pure。

例子：

```
pragma solidity ^0.5.0;

contract Calculator {
    function add(uint a, uint b) public pure returns (uint) {
 return a + b;
    }
}
```

在上面的合约中，`add` 函数将两个参数相加，并返回它们的和。由于这是一个 pure 函数，因此并不会执行任何操作，也不会在合约存储中读取或写入任何数据，也不会读取区块链的任何数据。

一个 pure 函数也可以接受参数指定的字符串、bytes、uint、address、fixed 和 ufixed 类型，但是不能接受 memory 和 storage 类型。

### 其他示例
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

contract ViewAndPure {
uint public x = 1;

    // Promise not to modify the state.
    function addToX(uint y) public view returns (uint) {
        return x + y;
    }

    // Promise not to modify or read from the state.
    function add(uint i, uint j) public pure returns (uint) {
        return i + j;
    }
}
```

## 更多信息

以下是有关 view 和 pure 的更多信息：

- 在 Solidity 文档中查看 [函数调用视图](https://solidity.readthedocs.io/en/v0.5.0/contracts.html#view-functions)。
- 阅读 [Solidity 0.5.0 发布说明](https://solidity.readthedocs.io/en/v0.5.0/050-breaking-changes.html?highlight=view#changes-to-the-semantics-of-constant-functions) 以了解 constant 函数的更改。
- 查看 [view](https://solidity.readthedocs.io/en/v0.5.0/types.html#type-view) 和 [pure](https://solidity.readthedocs.io/en/v0.5.0/types.html#type-pure) 函数的文档。
