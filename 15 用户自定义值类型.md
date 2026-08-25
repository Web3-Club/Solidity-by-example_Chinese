# 用户自定义值类型

对应英文原页：https://solidity-by-example.org/user-defined-value-types

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

// 代码复制自 Optimism
// https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/src/dispute/lib/LibUDT.sol

type Duration is uint64;

type Timestamp is uint64;

type Clock is uint128;

library LibClock {
    function wrap(Duration _duration, Timestamp _timestamp)
        internal
        pure
        returns (Clock clock_)
    {
        assembly {
            // data | Duration | Timestamp
            // bit  | 0 ... 63 | 64 ... 127
            clock_ := or(shl(0x40, _duration), _timestamp)
        }
    }

    function duration(Clock _clock)
        internal
        pure
        returns (Duration duration_)
    {
        assembly {
            duration_ := shr(0x40, _clock)
        }
    }

    function timestamp(Clock _clock)
        internal
        pure
        returns (Timestamp timestamp_)
    {
        assembly {
            timestamp_ := shr(0xC0, shl(0xC0, _clock))
        }
    }
}

// 不使用用户自定义值类型的 Clock 库
library LibClockBasic {
    function wrap(uint64 _duration, uint64 _timestamp)
        internal
        pure
        returns (uint128 clock)
    {
        assembly {
            clock := or(shl(0x40, _duration), _timestamp)
        }
    }
}

contract Examples {
    function example_no_uvdt() external view {
        // 不使用 UDVT（用户自定义值类型）
        uint128 clock;
        uint64 d = 1;
        uint64 t = uint64(block.timestamp);
        clock = LibClockBasic.wrap(d, t);
        // 哎呀！输入顺序错了，但仍然能编译
        clock = LibClockBasic.wrap(t, d);
    }

    function example_uvdt() external view {
        // 将值类型转换为用户自定义值类型
        Duration d = Duration.wrap(1);
        Timestamp t = Timestamp.wrap(uint64(block.timestamp));
        // 将用户自定义值类型转换回原始值类型
        uint64 d_u64 = Duration.unwrap(d);
        uint64 t_u64 = Timestamp.unwrap(t);

        // LibClock 示例
        Clock clock = Clock.wrap(0);
        clock = LibClock.wrap(d, t);
        // 哎呀！输入顺序错了
        // 这样将无法编译
        // clock = LibClock.wrap(t, d);
    }
}
```

---
## 关注我们
[Yanbo的Twitter](https://x.com/Yanbo2004)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)


[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
