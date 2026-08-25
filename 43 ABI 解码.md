# ABI 解码

对应英文原页：https://solidity-by-example.org/abi-decode

`abi.encode` 把数据编码成 `bytes`。

`abi.decode` 把 `bytes` 解码回数据。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

contract AbiDecode {
    struct MyStruct {
        string name;
        uint256[2] nums;
    }

    function encode(
        uint256 x,
        address addr,
        uint256[] calldata arr,
        MyStruct calldata myStruct
    ) external pure returns (bytes memory) {
        return abi.encode(x, addr, arr, myStruct);
    }

    function decode(bytes calldata data)
        external
        pure
        returns (
            uint256 x,
            address addr,
            uint256[] memory arr,
            MyStruct memory myStruct
        )
    {
        // (uint x, address addr, uint[] memory arr, MyStruct myStruct) = ...
        (x, addr, arr, myStruct) =
            abi.decode(data, (uint256, address, uint256[], MyStruct));
    }
}
```

---
## 关注我们
[Yanbo的Twitter](https://x.com/Yanbo2004)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)


[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
