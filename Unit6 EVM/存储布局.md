# EVM 存储布局

对应英文原页：https://solidity-by-example.org/evm/storage

示例如下

- 使用 `assembly` 读写存储
- 不同数据类型的存储布局

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

// Yul —— 用于 Solidity 内联汇编的语言
contract YulIntro {
    // Yul 赋值
    function test_yul_var() public pure returns (uint256) {
        uint256 s = 0;

        assembly {
            // 声明变量
            let x := 1
            // 重新赋值
            x := 2
            // 赋给 Solidity 变量
            s := 2
        }

        return s;
    }

    // Yul 类型（一切都是 bytes32）
    function test_yul_types()
        public
        pure
        returns (bool x, uint256 y, bytes32 z)
    {
        assembly {
            x := 1
            y := 0xaaa
            z := "Hello Yul"
        }

        return (x, y, z);
    }
}

contract EVMStorageSingleSlot {
    // EVM 存储
    // 2**256 个槽，每个槽最多存储 32 字节
    // 槽按状态变量声明的顺序分配
    // 小于 32 字节的数据会打包进同一个槽（从右到左）
    // sstore(k, v) = 将 v 存入槽 k
    // sload(k) = 从槽 k 加载 32 字节

    // 单个变量存储在一个槽中
    // 槽 0
    uint256 public s_x;
    // 槽 1
    uint256 public s_y;
    // 槽 2
    bytes32 public s_z;

    function test_sstore() public {
        assembly {
            sstore(0, 111)
            sstore(1, 222)
            sstore(2, 0xababab)
        }
    }

    function test_sstore_again() public {
        // 使用 .slot 访问槽
        assembly {
            sstore(s_x.slot, 123)
            sstore(s_y.slot, 456)
            sstore(s_z.slot, 0xcdcdcd)
        }
    }

    function test_sload()
        public
        view
        returns (uint256 x, uint256 y, bytes32 z)
    {
        assembly {
            x := sload(0)
            y := sload(1)
            z := sload(2)
        }

        return (x, y, z);
    }

    function test_sload_again()
        public
        view
        returns (uint256 x, uint256 y, bytes32 z)
    {
        assembly {
            x := sload(s_x.slot)
            y := sload(s_y.slot)
            z := sload(s_z.slot)
        }

        return (x, y, z);
    }
}

contract EVMStoragePackedSlotBytes {
    // 槽 0（从右到左打包）
    bytes4 public b4 = 0xabababab;
    bytes2 public b2 = 0xcdcd;

    function get() public view returns (bytes32 b32) {
        assembly {
            b32 := sload(0)
        }
    }
}

contract BitMasking {
    function test_mask() public pure returns (bytes32 mask) {
        assembly {
            // |       256 bits        |
            // 000 ... 000 | 111 ... 111
            //             | 16 bits
            // 0x000000000000000000000000000000000000000000000000000000000000ffff
            mask := sub(shl(16, 1), 1)
        }
    }

    function test_shift_mask() public pure returns (bytes32 mask) {
        assembly {
            // |               256 bits                |
            // 000 ... 000 | 111 ... 111 | 000 ... 000 |
            //             | 16 bits     | 32 bits
            // 0x0000000000000000000000000000000000000000000000000000ffff00000000
            mask := shl(32, sub(shl(16, 1), 1))
        }
    }

    function test_not_mask() public pure returns (bytes32 mask) {
        assembly {
            // |               256 bits                |
            // 111 ... 111 | 000 ... 000 | 111 ... 111 |
            //             | 16 bits     | 32 bits
            // 0xffffffffffffffffffffffffffffffffffffffffffffffffffff0000ffffffff
            mask := not(shl(32, sub(shl(16, 1), 1)))
        }
    }
}

contract EVMStoragePackedSlot {
    // 小于 32 字节的数据会打包进同一个槽
    // 位掩码（如何构造 111...111）
    // slot, offset

    // 槽 0
    uint128 public s_a;
    uint64 public s_b;
    uint32 public s_c;
    uint32 public s_d;
    // 槽 1
    // 20 字节 = 160 位
    address public s_addr;
    // 96 位
    uint64 public s_x;
    uint32 public s_y;

    function test_sstore() public {
        assembly {
            // 从槽 0 加载 32 字节
            let v := sload(0)

            // s_d | s_c | s_b | s_a
            // 32  | 32  | 64  | 128 bits

            // 设置 s_a = 11
            // mask = 从右边数第 128 位及其左侧全为 1
            //        111 ... 111 | 000 ... 000
            //                    |    128 bits
            let mask_a := not(sub(shl(128, 1), 1))
            // 将最左侧 128 位设为 0
            v := and(v, mask_a)
            // 设置 s_a = 11
            v := or(v, 11)

            // 设置 s_b = 22
            // mask = 111...111 | 000 ... 000 | 111 ... 111
            //                  |     64 bits |    128 bits
            let mask_b := not(shl(128, sub(shl(64, 1), 1)))
            // 将第 128 到 191 位清零，清除 s_b 的旧值
            v := and(v, mask_b)
            v := or(v, shl(128, 22))

            // 设置 s_c = 33
            // mask = 111...111 | 000...000 | 111 ... 111 | 111 ... 111
            //                  |   32 bits |     64 bits |    128 bits
            let mask_c := not(shl(192, sub(shl(32, 1), 1)))
            // 将第 192 到 223 位清零，清除 s_c 的旧值
            v := and(v, mask_c)
            v := or(v, shl(192, 33))

            // 设置 s_d = 44
            // mask = 000...000 | 111...111 | 111 ... 111 | 111 ... 111
            //                  |   32 bits |     64 bits |    128 bits
            let mask_d := not(shl(224, sub(shl(32, 1), 1)))
            // 将第 224 到 255 位清零，清除 s_d 的旧值
            v := and(v, mask_d)
            v := or(v, shl(224, 44))

            // 将新值存入槽 0
            sstore(0, v)
        }
    }

    function test_slot_0_offset()
        public
        pure
        returns (
            uint256 a_offset,
            uint256 b_offset,
            uint256 c_offset,
            uint256 d_offset
        )
    {
        // a_offset =  0 =  0 * 8 =   0 bits
        // b_offset = 16 = 16 * 8 = 128 bits
        // c_offset = 24 = 24 * 8 = 192 bits
        // d_offset = 28 = 28 * 8 = 224 bits
        assembly {
            a_offset := s_a.offset
            b_offset := s_b.offset
            c_offset := s_c.offset
            d_offset := s_d.offset
        }
    }

    function test_slot_1_offset()
        public
        pure
        returns (uint256 addr_offset, uint256 x_offset, uint256 y_offset)
    {
        // addr_offset = 0
        // x_offset = 20
        // y_offset = 28
        assembly {
            addr_offset := s_addr.offset
            x_offset := s_x.offset
            y_offset := s_y.offset
        }
    }

    // slot 和 offset
    function test_sstore_using_offset() public {
        // a_offset =  0 =  0 * 8 =   0 bits
        // b_offset = 16 = 16 * 8 = 128 bits
        // c_offset = 24 = 24 * 8 = 192 bits
        // d_offset = 28 = 28 * 8 = 224 bits
        assembly {
            // 从槽 0 加载 32 字节
            let v := sload(s_a.slot)

            // s_d | s_c | s_b | s_a
            // 32  | 32  | 64  | 128 bits

            // 设置 s_a = 111
            // mask = 从右边数第 128 位及其左侧全为 1
            //        111 ... 111 | 000 ... 000
            //                    |    128 bits
            let mask_a := not(sub(shl(128, 1), 1))
            // 将最左侧 128 位设为 0
            v := and(v, mask_a)
            // 设置 s_a = 1
            v := or(v, 111)

            // 设置 s_b = 222
            // mask = 111...111 | 000 ... 000 | 111 ... 111
            //                  |     64 bits |    128 bits
            let mask_b := not(shl(mul(s_b.offset, 8), sub(shl(64, 1), 1)))
            // 将第 128 到 191 位清零，清除 s_b 的旧值
            v := and(v, mask_b)
            v := or(v, shl(mul(s_b.offset, 8), 222))

            // 设置 s_c = 333
            // mask = 111...111 | 000...000 | 111 ... 111 | 111 ... 111
            //                  |   32 bits |     64 bits |    128 bits
            let mask_c := not(shl(mul(s_c.offset, 8), sub(shl(32, 1), 1)))
            // 将第 192 到 223 位清零，清除 s_c 的旧值
            v := and(v, mask_c)
            v := or(v, shl(mul(s_c.offset, 8), 333))

            // 设置 s_d = 444
            // mask = 000...000 | 111...111 | 111 ... 111 | 111 ... 111
            //                  |   32 bits |     64 bits |    128 bits
            let mask_d := not(shl(mul(s_d.offset, 8), sub(shl(32, 1), 1)))
            // 将第 224 到 255 位清零，清除 s_d 的旧值
            v := and(v, mask_d)
            v := or(v, shl(mul(s_d.offset, 8), 444))

            // 将新值存入槽 0
            sstore(s_a.slot, v)
        }
    }
}

contract EVMStorageStruct {
    struct SingleSlot {
        uint128 x;
        uint64 y;
        uint64 z;
    }

    struct MultipleSlots {
        uint256 a;
        uint256 b;
        uint256 c;
    }

    // 槽 0
    SingleSlot public single = SingleSlot({x: 1, y: 2, z: 3});
    // 槽 1、2、3
    MultipleSlots public multi = MultipleSlots({a: 11, b: 22, c: 33});

    function test_get_single_slot_struct()
        public
        view
        returns (uint128 x, uint64 y, uint64 z)
    {
        assembly {
            let s := sload(0)
            //  z |  y | x
            // 64 | 64 | 128 bits
            // 类型转换会截掉左侧的位
            x := s
            y := shr(128, s)
            z := shr(192, s)
        }
    }

    function test_get_multiple_slots_struct()
        public
        view
        returns (uint256 a, uint256 b, uint256 c)
    {
        assembly {
            a := sload(1)
            b := sload(2)
            c := sload(3)
        }
    }
}

contract EVMStorageConstants {
    // 槽 0
    uint256 public s0 = 1;
    // 常量和不可变量不使用存储
    uint256 public constant X = 123;
    address public immutable owner;
    // 槽 1
    uint256 public s1 = 2;

    constructor() {
        owner = msg.sender;
    }

    function test_get_slots() public view returns (uint256 v0, uint256 v1) {
        assembly {
            v0 := sload(0)
            v1 := sload(1)
        }
    }
}

contract EVMStorageFixedArray {
    // 元素为 32 字节的定长数组，元素槽位 = 数组声明所在槽 + 数组元素下标
    // 槽 0、1、2
    uint256[3] private arr_0 = [1, 2, 3];
    // 槽 3、4、5
    uint256[3] private arr_1 = [4, 5, 6];
    // 槽 + 打包数据的下标
    // 槽 6、6、7、7、8
    uint128[5] private arr_2 = [7, 8, 9, 10, 11];

    function test_arr_0(uint256 i) public view returns (uint256 v) {
        assembly {
            // arr_0 从槽 0 开始
            v := sload(add(0, i))
        }
    }

    function test_arr_1(uint256 i) public view returns (uint256 v) {
        assembly {
            // arr_1 从槽 3 开始
            v := sload(add(3, i))
        }
    }

    function test_arr_2(uint256 i) public view returns (uint128 v) {
        assembly {
            // arr_2 从槽 6 开始
            let b32 := sload(add(6, div(i, 2)))
            // 槽 6 = 第 1 个元素 | 第 0 个元素
            // 槽 7 = 第 3 个元素 | 第 2 个元素
            // 槽 8 = 000 ... 000 | 第 4 个元素

            // i 为偶数 => 取右侧 128 位 => 将 bytes32 转为 uint128（截掉左侧 128 位）
            // i 为奇数  => 取左侧 128 位  => 右移 128 位

            switch mod(i, 2)
            case 1 { v := shr(128, b32) }
            default { v := b32 }
        }
    }
}

contract EVMStorageDynamicArray {
    // 元素槽位 = keccak256(该数组声明所在槽) + 元素大小 * 元素下标
    // keccak256(0) + 1 * index
    uint256[] private arr = [11, 22, 33];
    // keccak256(1) + 1 / 2 * index
    uint128[] private arr_2 = [1, 2, 3];

    function test_arr(uint256 slot, uint256 i)
        public
        view
        returns (uint256 v, bytes32 b32, uint256 len)
    {
        bytes32 start = keccak256(abi.encode(slot));

        assembly {
            len := sload(slot)
            v := sload(add(start, i))
            b32 := v
        }
    }
}

contract EVMStorageMapping {
    // 值的槽位 = keccak256(key, mapping 声明所在槽)
    mapping(address => uint256) public map;

    address public constant ADDR_1 = address(1);
    address public constant ADDR_2 = address(2);
    address public constant ADDR_3 = address(3);

    constructor() {
        map[ADDR_1] = 11;
        map[ADDR_2] = 22;
        map[ADDR_3] = 33;
    }

    function test_mapping(address key) public view returns (uint256 v) {
        uint256 slot = 0;
        bytes32 slot_v = keccak256(abi.encode(key, slot));

        assembly {
            v := sload(slot_v)
        }
    }
}

contract EVMStorageNestedMapping {
    // key0 => key1 => val
    // 值的槽位 = keccak256(key1, keccak256(key0, 嵌套 mapping 声明所在槽))
    mapping(address => mapping(address => uint256)) public map;

    address public constant ADDR_1 = address(1);
    address public constant ADDR_2 = address(2);
    address public constant ADDR_3 = address(3);

    constructor() {
        map[ADDR_1][ADDR_2] = 11;
        map[ADDR_2][ADDR_3] = 22;
        map[ADDR_3][ADDR_1] = 33;
    }

    function test_nested_mapping(address key_0, address key_1)
        public
        view
        returns (uint256 v)
    {
        uint256 slot = 0;
        bytes32 s0 = keccak256(abi.encode(key_0, slot));
        bytes32 s1 = keccak256(abi.encode(key_1, s0));

        assembly {
            v := sload(s1)
        }
    }
}

contract EVMStorageMappingArray {
    // mapping 中值的槽位 = keccak256(key, slot)
    // 数组元素槽位 = keccak256(slot) + index
    // mapping -> array -> keccak256(keccak256(key, mapping 声明所在槽)) + index
    mapping(address => uint256[]) public map;

    address public constant ADDR_1 = address(1);
    address public constant ADDR_2 = address(2);

    constructor() {
        map[ADDR_1].push(11);
        map[ADDR_1].push(22);
        map[ADDR_1].push(33);
        map[ADDR_2].push(44);
        map[ADDR_2].push(55);
        map[ADDR_2].push(66);
    }

    function test_map_arr(address addr, uint256 i)
        public
        view
        returns (uint256 v, uint256 len)
    {
        uint256 map_slot = 0;
        bytes32 map_hash = keccak256(abi.encode(addr, map_slot));
        bytes32 arr_hash = keccak256(abi.encode(map_hash));

        assembly {
            len := sload(map_hash)
            v := sload(add(arr_hash, i))
        }
    }
}

contract EVMStorageDynamicArrayStruct {
    struct Point {
        uint256 x;
        uint128 y;
        uint128 z;
    }

    // 元素槽位 = keccak256(该数组声明所在槽) + 元素下标
    // keccak256(0) + index * struct 大小
    Point[] private arr;

    constructor() {
        arr.push(Point(11, 22, 33));
        arr.push(Point(44, 55, 66));
        arr.push(Point(77, 88, 99));
    }

    function test_struct_arr(uint256 i)
        public
        view
        returns (uint256 x, uint128 y, uint128 z, uint256 len)
    {
        uint256 slot = 0;
        bytes32 start = keccak256(abi.encode(slot));

        assembly {
            len := sload(slot)
            // s0 = keccak256(0)
            // index | slot        | values
            //     0 | slot s0 + 0 | arr[0].x
            //     0 | slot s0 + 1 | arr[0].z | arr[0].y
            //     1 | slot s0 + 2 | arr[1].x
            //     1 | slot s0 + 3 | arr[1].z | arr[1].y
            //     2 | slot s0 + 4 | arr[2].x
            //     2 | slot s0 + 5 | arr[2].z | arr[2].y
            x := sload(add(start, mul(i, 2)))
            let zy := sload(add(start, add(mul(i, 2), 1)))
            // uint128 会从 32 字节中截掉最左侧 128 位
            y := zy
            z := shr(128, zy)
        }
    }
}

```

---
## 关注我们
[Yanbo的Twitter](https://x.com/Yanbo2004)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)


[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
