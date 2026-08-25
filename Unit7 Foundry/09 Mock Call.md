# Mock Call

对应英文原页：https://solidity-by-example.org/foundry/mock-call

使用 `mockCall` 设置函数调用的返回值。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

import "forge-std/Test.sol";

contract Target {
    function f(uint256 x, uint256 y) external view returns (uint256) {
        return g();
    }

    function g() internal view returns (uint256) {
        return 1;
    }
}

contract MockCallTest is Test {
    Target target;

    function setUp() public {
        target = new Target();
    }

    function test() public {
        uint256 x = 1;
        uint256 y = 1;
        vm.mockCall(
            address(target),
            abi.encodeCall(Target.f, (x, y)),
            abi.encode(uint256(99))
        );

        // 返回 99
        uint256 res = target.f(x, y);
        console.log("res", res);
    }
}

```

---
## 关注我们
[Yanbo的Twitter](https://x.com/Yanbo2004)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)


[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
