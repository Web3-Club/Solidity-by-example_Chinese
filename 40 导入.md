# 导入

对应英文原页：https://solidity-by-example.org/import

在 Solidity 中可以导入本地文件和外部文件。

### 本地

下面是我们的文件夹结构。

```
├── Import.sol
└── Foo.sol
```

Foo.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

struct Point {
    uint256 x;
    uint256 y;
}

error Unauthorized(address caller);

function add(uint256 x, uint256 y) pure returns (uint256) {
    return x + y;
}

contract Foo {
    string public name = "Foo";
}
```

Import.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

// 从当前目录导入 Foo.sol
import "./Foo.sol";

// import {symbol1 as alias, symbol2} from "filename";
import {Unauthorized, add as func, Point} from "./Foo.sol";

contract Import {
    // 初始化 Foo.sol
    Foo public foo = new Foo();

    // 通过获取 name 来测试 Foo.sol。
    function getFooName() public view returns (string memory) {
        return foo.name();
    }
}
```

### 外部

也可以从 [GitHub](https://github.com) 导入，只需复制 url 即可

```solidity
// https://github.com/owner/repo/blob/branch/path/to/Contract.sol
import "https://github.com/owner/repo/blob/branch/path/to/Contract.sol";

// 示例：从 openzeppelin-contract 仓库的 release-v4.5 分支导入 ECDSA.sol
// https://github.com/OpenZeppelin/openzeppelin-contracts/blob/release-v4.5/contracts/utils/cryptography/ECDSA.sol
import "https://github.com/OpenZeppelin/openzeppelin-contracts/blob/release-v4.5/contracts/utils/cryptography/ECDSA.sol";
```

---
## 关注我们
[Yanbo的Twitter](https://x.com/Yanbo2004)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)


[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
