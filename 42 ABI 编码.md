# ABI 编码

对应英文原页：https://solidity-by-example.org/abi-encode

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

interface IERC20 {
    function transfer(address, uint256) external;
}

contract Token {
    function transfer(address, uint256) external {}
}

contract AbiEncode {
    function test(address _contract, bytes calldata data) external {
        (bool ok,) = _contract.call(data);
        require(ok, "call failed");
    }

    function encodeWithSignature(address to, uint256 amount)
        external
        pure
        returns (bytes memory)
    {
        // 不会检查拼写错误 - "transfer(address, uint)"
        return abi.encodeWithSignature("transfer(address,uint256)", to, amount);
    }

    function encodeWithSelector(address to, uint256 amount)
        external
        pure
        returns (bytes memory)
    {
        // 不会检查类型 - (IERC20.transfer.selector, true, amount)
        return abi.encodeWithSelector(IERC20.transfer.selector, to, amount);
    }

    function encodeCall(address to, uint256 amount)
        external
        pure
        returns (bytes memory)
    {
        // 拼写错误和类型错误将无法通过编译
        return abi.encodeCall(IERC20.transfer, (to, amount));
    }
}
```

---
## 关注我们
[Yanbo的Twitter](https://twitter.com/YanboOfficial)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)

加入Web3Club 官方讨论群：YanboTravelAllWorld（微信号）

[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
