# 汇编中的二进制求幂

对应英文原页：https://solidity-by-example.org/app/assembly-bin-exp

`assembly` 中二进制求幂的示例

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

contract AssemblyBinExp {
    // 用二进制求幂计算 x**n
    function rpow(uint256 x, uint256 n, uint256 b)
        public
        pure
        returns (uint256 z)
    {
        assembly {
            switch x
            // x = 0
            case 0 {
                switch n
                // n = 0 --> x**n = 0**0 --> 1
                case 0 { z := b }
                // n > 0 --> x**n = 0**n --> 0
                default { z := 0 }
            }
            default {
                switch mod(n, 2)
                // x > 0 且 n 为偶数 --> z = 1
                case 0 { z := b }
                // x > 0 且 n 为奇数 --> z = x
                default { z := x }

                let half := div(b, 2) // 用于四舍五入。
                // n = n / 2, while n > 0, n = n / 2
                for { n := div(n, 2) } n { n := div(n, 2) } {
                    let xx := mul(x, x)
                    // 检查溢出：若 xx / x != x 则 revert
                    if iszero(eq(div(xx, x), x)) { revert(0, 0) }
                    // 四舍五入 (xx + half) / b
                    let xxRound := add(xx, half)
                    // 检查溢出：若 xxRound < xx 则 revert
                    if lt(xxRound, xx) { revert(0, 0) }
                    x := div(xxRound, b)
                    // 如果 n % 2 == 1
                    if mod(n, 2) {
                        let zx := mul(z, x)
                        // 若 x != 0 且 zx / x != z 则 revert
                        if and(iszero(iszero(x)), iszero(eq(div(zx, x), z))) {
                            revert(0, 0)
                        }
                        // 四舍五入 (zx + half) / b
                        let zxRound := add(zx, half)
                        // 检查溢出：若 zxRound < zx 则 revert
                        if lt(zxRound, zx) { revert(0, 0) }
                        z := div(zxRound, b)
                    }
                }
            }
        }
    }
}
```

---
## 关注我们
[Yanbo的Twitter](https://x.com/Yanbo2004)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)


[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
