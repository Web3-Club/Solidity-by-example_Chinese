# ABI Decode解码

`abi.encode` 将数据编码为 `bytes` 格式。

`abi.decode` 将 `bytes` 解码回原始数据。

```solidity
// SPDX-License-Identifier: MIT  // SPDX 许可证标识符：MIT 许可证
pragma solidity ^0.8.26;  // 版本指令，要求使用 Solidity 0.8.26 及以上版本

// AbiDecode 合约：演示 ABI 编码和解码
contract AbiDecode {  // 定义 AbiDecode 合约
    // MyStruct 结构体：自定义复杂数据类型
    struct MyStruct {  // struct 定义包含多种类型字段的结构体
        string name;         // 字符串名称
        uint256[2] nums;     // 长度为 2 的 uint256 定长数组
    }

    // encode 函数：将多种类型的数据编码为 bytes
    function encode(  // 接收四种不同类型的参数
        uint256 x,                        // uint256 类型
        address addr,                     // address 类型
        uint256[] calldata arr,           // uint256 动态数组（calldata 位置）
        MyStruct calldata myStruct        // 自定义结构体（calldata 位置）
    ) external pure returns (bytes memory) {  // 返回编码后的 bytes
        // abi.encode 将所有参数紧密打包编码为 bytes
        return abi.encode(x, addr, arr, myStruct);  // 编码所有参数
    }

    // decode 函数：将 bytes 解码回原始数据类型
    function decode(bytes calldata data)  // 接收编码后的 bytes 数据
        external
        pure
        returns (
            uint256 x,                    // 解码出的 uint256
            address addr,                 // 解码出的地址
            uint256[] memory arr,         // 解码出的动态数组（memory 位置）
            MyStruct memory myStruct      // 解码出的结构体（memory 位置）
        )
    {
        // 以下注解展示了解构赋值的等效写法：
        // (uint x, address addr, uint[] memory arr, MyStruct myStruct) = ...

        // abi.decode 将 bytes 解码回原始类型
        // 必须指定与编码时完全一致的类型列表
        (x, addr, arr, myStruct) =
            abi.decode(data, (uint256, address, uint256[], MyStruct));  // 按指定类型解码 data
    }
}
```
