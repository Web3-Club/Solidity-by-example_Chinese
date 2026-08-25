# 使用 Create2 预计算合约地址

对应英文原页：https://solidity-by-example.org/app/create2

可以使用 `create2` 在合约部署之前预计算合约地址

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

contract Factory {
    // 返回新部署合约的地址
    function deploy(address _owner, uint256 _foo, bytes32 _salt)
        public
        payable
        returns (address)
    {
        // 这是一种不使用汇编调用 create2 的较新语法，只需传入 salt
        // https://docs.soliditylang.org/en/latest/control-structures.html#salted-contract-creations-create2
        return address(new TestContract{salt: _salt}(_owner, _foo));
    }
}

// 这是使用汇编的旧方法
contract FactoryAssembly {
    event Deployed(address addr, uint256 salt);

    // 1. 获取要部署的合约的字节码
    // 注意：_owner 和 _foo 是 TestContract 构造函数的参数
    function getBytecode(address _owner, uint256 _foo)
        public
        pure
        returns (bytes memory)
    {
        bytes memory bytecode = type(TestContract).creationCode;

        return abi.encodePacked(bytecode, abi.encode(_owner, _foo));
    }

    // 2. 计算要部署的合约的地址
    // 注意：_salt 是用于创建地址的随机数
    function getAddress(bytes memory bytecode, uint256 _salt)
        public
        view
        returns (address)
    {
        bytes32 hash = keccak256(
            abi.encodePacked(
                bytes1(0xff), address(this), _salt, keccak256(bytecode)
            )
        );

        // 注意：将哈希的最后 20 个字节转换为地址
        return address(uint160(uint256(hash)));
    }

    // 3. 部署合约
    // 注意：
    // 检查事件日志 Deployed，其中包含已部署的 TestContract 的地址。
    // 日志中的地址应等于上面计算得到的地址。
    function deploy(bytes memory bytecode, uint256 _salt) public payable {
        address addr;

        /*
        注意：如何调用 create2

        create2(v, p, n, s)
        用内存 p 到 p + n 处的代码创建新合约
        并发送 v wei
        然后返回新地址
        其中新地址 = keccak256(0xff + address(this) + s + keccak256(mem[p…(p+n))) 的前 20 个字节
              s = 大端序 256 位值
        */
        assembly {
            addr :=
                create2(
                    callvalue(), // 当前调用附带的 wei
                    // 跳过前 32 字节后才是实际代码
                    add(bytecode, 0x20),
                    mload(bytecode), // 加载前 32 字节中包含的代码大小
                    _salt // 函数参数中的 salt
                )

            if iszero(extcodesize(addr)) { revert(0, 0) }
        }

        emit Deployed(addr, _salt);
    }
}

contract TestContract {
    address public owner;
    uint256 public foo;

    constructor(address _owner, uint256 _foo) payable {
        owner = _owner;
        foo = _foo;
    }

    function getBalance() public view returns (uint256) {
        return address(this).balance;
    }
}
```

---
## 关注我们
[Yanbo的Twitter](https://x.com/Yanbo2004)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)


[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
