# Store

对应英文原页：https://solidity-by-example.org/foundry/vm-store

使用 `vm.store` 在测试中直接写入合约的存储槽。

这适用于：

- 无需调用合约函数即可设置测试场景
- 绕过访问控制以测试特定状态
- 测试正常情况下难以到达的边界情况

对于 mapping，用 `keccak256(abi.encode(key, mappingSlot))` 计算槽位。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

import {Test, console2} from "forge-std/Test.sol";

contract Vault {
    // 槽 0
    address public owner;
    // 槽 1
    uint256 public password;
    // 槽 2
    mapping(address => uint256) public balances;

    constructor(uint256 _password) {
        owner = msg.sender;
        password = _password;
    }

    function withdraw() external {
        require(msg.sender == owner, "not owner");
        uint256 bal = balances[msg.sender];
        balances[msg.sender] = 0;
        payable(msg.sender).transfer(bal);
    }
}

contract StoreTest is Test {
    Vault vault;

    function setUp() public {
        vault = new Vault(12345);
    }

    // vm.store(address account, bytes32 slot, bytes32 value)
    // - account: 合约地址
    // - slot: 要写入的存储槽
    // - value: 要写入的值

    function test_store_simple_slot() public {
        // 槽 0 存储 owner 地址
        // 将 owner 改为 address(1)
        vm.store(address(vault), bytes32(uint256(0)), bytes32(uint256(uint160(address(1)))));
        assertEq(vault.owner(), address(1));

        // 槽 1 存储 password
        // 将 password 改为 999
        vm.store(address(vault), bytes32(uint256(1)), bytes32(uint256(999)));
        assertEq(vault.password(), 999);
    }

    function test_store_mapping() public {
        // 对于 mapping，槽位计算为：
        // keccak256(abi.encode(key, mapping_slot))

        address user = address(0xBEEF);
        uint256 mappingSlot = 2; // balances 在槽 2

        // 计算 balances[user] 的存储槽
        bytes32 slot = keccak256(abi.encode(user, mappingSlot));

        // 将 user 的余额设为 100 ether
        vm.store(address(vault), slot, bytes32(uint256(100 ether)));

        assertEq(vault.balances(user), 100 ether);
    }
}

```

---
## 关注我们
[Yanbo的Twitter](https://twitter.com/YanboOfficial)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)

加入Web3Club 官方讨论群：YanboTravelAllWorld（微信号）

[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
