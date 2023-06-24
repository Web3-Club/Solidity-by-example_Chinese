# CREATE2

`CREATE2` 在以太坊上创建合约时比 `CREATE` 更为灵活。

一个常见的用例是在同一地址下多次部署合约（例如，在 Uniswap 中，工厂合约被部署多次）。使用 `CREATE2`，可以预测新合约的地址，因此可以更有效地与其交互。

## 如何获得 CREATE2 地址

假设我们有三个参数：`_salt`、`_bytecode` 和 `_sender_address`：

1. 通过将 `_salt`、`_bytecode` 和 `_sender_address` 连接起来并计算它们的哈希值，生成一个哈希。
2. 将哈希值转换为地址。这是通过将哈希值的最后 20 个字节作为地址获取的。
3. 如果该地址上还没有合约，则可以通过调用 `CREATE2` 创建一个新合约。如果已经存在合约，则可以直接与其交互。

```solidity
function computeAddress(bytes32 _salt, bytes memory _bytecode, address _sender_address) public view returns (address) {
    bytes32 hash = keccak256(abi.encodePacked(
        bytes1(0xff),
        _sender_address,
        _salt,
        keccak256(_bytecode)
    ));

    return address(uint160(uint256(hash)));
}
```

## 示例

以下是如何使用 `CREATE2` 部署合约的示例。

```solidity
pragma solidity ^0.5.16;

contract Counter {
    uint256 public count;

    constructor() public {
        count = 0;
    }

    function increment() public {
        count += 1;
    }
}

contract CounterFactory {
    event Created(address counter);

    function createCounter(bytes32 _salt) external {
        Counter counter = new Counter{salt: _salt}();
        emit Created(address(counter));
    }
}
```

在这个示例中，`CounterFactory` 可以通过调用 `createCounter` 在同一地址下创建多个 `Counter` 合约。使用 `CREATE2`，可以预测新合约的地址，因此可以与其交互而不必先查询它是否存在。