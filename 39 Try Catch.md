# Try / Catch

对应英文原页：https://solidity-by-example.org/try-catch

`try / catch` 只能捕获来自外部函数调用和合约创建的错误。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

// 用于 try / catch 示例的外部合约
contract Foo {
    address public owner;

    constructor(address _owner) {
        require(_owner != address(0), "invalid address");
        assert(_owner != 0x0000000000000000000000000000000000000001);
        owner = _owner;
    }

    function myFunc(uint256 x) public pure returns (string memory) {
        require(x != 0, "require failed");
        return "my func was called";
    }
}

contract Bar {
    event Log(string message);
    event LogBytes(bytes data);

    Foo public foo;

    constructor() {
        // 此 Foo 合约用于演示带外部调用的 try catch 示例
        foo = new Foo(msg.sender);
    }

    // 带外部调用的 try / catch 示例
    // tryCatchExternalCall(0) => Log("external call failed")
    // tryCatchExternalCall(1) => Log("my func was called")
    function tryCatchExternalCall(uint256 _i) public {
        try foo.myFunc(_i) returns (string memory result) {
            emit Log(result);
        } catch {
            emit Log("external call failed");
        }
    }

    // 带合约创建的 try / catch 示例
    // tryCatchNewContract(0x0000000000000000000000000000000000000000) => Log("invalid address")
    // tryCatchNewContract(0x0000000000000000000000000000000000000001) => LogBytes("")
    // tryCatchNewContract(0x0000000000000000000000000000000000000002) => Log("Foo created")
    function tryCatchNewContract(address _owner) public {
        try new Foo(_owner) returns (Foo foo) {
            // 可以在这里使用变量 foo
            emit Log("Foo created");
        } catch Error(string memory reason) {
            // 捕获失败的 revert() 和 require()
            emit Log(reason);
        } catch (bytes memory reason) {
            // 捕获失败的 assert()
            emit LogBytes(reason);
        }
    }
}
```

---
## 关注我们
[Yanbo的Twitter](https://x.com/Yanbo2004)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)


[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
