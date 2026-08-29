# Call

对应英文原页：https://solidity-by-example.org/call

`call` 是用于与其他合约交互的低级函数。

当你只是通过调用 `fallback` 函数发送以太币时，推荐使用这种方法。

然而，它并不是调用已有函数的推荐方式。

### 不推荐使用低级 call 的几个原因

- revert 不会向上冒泡
- 类型检查会被绕过
- 省略了对函数是否存在的检查

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

contract Receiver {
    event Received(address caller, uint256 amount, string message);

    fallback() external payable {
        emit Received(msg.sender, msg.value, "Fallback was called");
    }

    function foo(string memory _message, uint256 _x)
        public
        payable
        returns (uint256)
    {
        emit Received(msg.sender, msg.value, _message);

        return _x + 1;
    }
}

contract Caller {
    event Response(bool success, bytes data);

    // 假设 Caller 合约没有 Receiver 合约的源代码，
    // 但我们知道 Receiver 合约的地址以及要调用的函数。
    function testCallFoo(address payable _addr) public payable {
        // 你可以发送以太币并指定自定义的 gas 数量
        (bool success, bytes memory data) = _addr.call{
            value: msg.value,
            gas: 5000
        }(abi.encodeWithSignature("foo(string,uint256)", "call foo", 123));

        emit Response(success, data);
    }

    // 调用不存在的函数会触发回退函数。
    function testCallDoesNotExist(address payable _addr) public payable {
        (bool success, bytes memory data) = _addr.call{value: msg.value}(
            abi.encodeWithSignature("doesNotExist()")
        );

        emit Response(success, data);
    }
}
```

---
## 关注我们
[Yanbo的Twitter](https://x.com/Yanbo2004)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)


[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
