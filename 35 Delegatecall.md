# Delegatecall

对应英文原页：https://solidity-by-example.org/delegatecall

`delegatecall` 是类似于 `call` 的低级函数。

当合约 `A` 对合约 `B` 执行 `delegatecall` 时，会执行 `B` 的代码，但使用的是合约 `A` 的存储、`msg.sender` 和 `msg.value`。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

// 注意：请先部署此合约
contract B {
    // 注意：存储布局必须与合约 A 相同
    uint256 public num;
    address public sender;
    uint256 public value;

    function setVars(uint256 _num) public payable {
        num = _num;
        sender = msg.sender;
        value = msg.value;
    }
}

contract A {
    uint256 public num;
    address public sender;
    uint256 public value;

    event DelegateResponse(bool success, bytes data);
    event CallResponse(bool success, bytes data);

    // 使用 delegatecall 的函数
    function setVarsDelegateCall(address _contract, uint256 _num)
        public
        payable
    {
        // 会修改 A 的存储；B 的存储不会被修改。
        (bool success, bytes memory data) = _contract.delegatecall(
            abi.encodeWithSignature("setVars(uint256)", _num)
        );

        emit DelegateResponse(success, data);
    }

    // 使用 call 的函数
    function setVarsCall(address _contract, uint256 _num) public payable {
        // 会修改 B 的存储；A 的存储不会被修改。
        (bool success, bytes memory data) = _contract.call{value: msg.value}(
            abi.encodeWithSignature("setVars(uint256)", _num)
        );

        emit CallResponse(success, data);
    }
}
```

---
## 关注我们
[Yanbo的Twitter](https://twitter.com/YanboOfficial)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)

加入Web3Club 官方讨论群：YanboTravelAllWorld（微信号）

[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
