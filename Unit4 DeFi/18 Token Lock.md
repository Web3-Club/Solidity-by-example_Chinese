# Token Lock

对应英文原页：https://solidity-by-example.org/defi/token-lock

在指定时间段内锁定代币并线性释放的合约示例。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

import {IERC20} from "./IERC20.sol";

contract Auth {
    mapping(address => bool) public authorized;

    modifier auth() {
        require(authorized[msg.sender], "not authorized");
        _;
    }

    constructor() {
        authorized[msg.sender] = true;
    }

    function allow(address user) external auth {
        authorized[user] = true;
    }

    function deny(address user) external auth {
        authorized[user] = false;
    }
}

contract TokenLock is Auth {
    struct Lock {
        // 锁定数量
        uint256 amount;
        // 上次锁定时间戳
        uint32 updatedAt;
        // 锁定到期时间戳
        uint32 expiresAt;
        // 锁定期
        uint32 duration;
    }

    mapping(address => Lock) public locks;
    mapping(address => uint256) public freed;

    function get(address token) external view returns (Lock memory) {
        return locks[token];
    }

    function set(address token, uint32 duration) external auth {
        locks[token].duration = duration;
    }

    function unlocked(address token) public view returns (uint256) {
        Lock memory l = locks[token];
        // block.timestamp >= expiresAt >= updatedAt
        // block.timestamp >= updateAt >= expiresAt
        if (block.timestamp >= l.expiresAt) {
            return l.amount;
        }
        // expiresAt > block.timestamp >= updateAt
        return l.amount * (block.timestamp - uint256(l.updatedAt))
            / uint256(l.expiresAt - l.updatedAt);
    }

    function claimable(address token) public view returns (uint256) {
        return freed[token] + unlocked(token);
    }

    function lock(address token, uint256 amount) external auth {
        uint256 free = unlocked(token);

        Lock storage l = locks[token];
        l.amount -= free;
        l.updatedAt = uint32(block.timestamp);
        l.expiresAt = uint32(block.timestamp) + l.duration;
        freed[token] += free;

        if (amount > 0) {
            IERC20(token).transferFrom(msg.sender, address(this), amount);
            l.amount += amount;
        }
    }

    function unlock(address token) external auth returns (uint256 amount) {
        uint256 free = unlocked(token);

        Lock storage l = locks[token];
        l.amount -= free;
        l.updatedAt = uint32(block.timestamp);

        amount = freed[token] + free;
        freed[token] = 0;

        if (amount > 0) {
            IERC20(token).transfer(msg.sender, amount);
        }
    }

    function sync(address token) external auth {
        uint256 reserved = freed[token] + locks[token].amount;
        uint256 bal = IERC20(token).balanceOf(address(this));
        if (bal > reserved) {
            IERC20(token).transfer(msg.sender, bal - reserved);
        }
    }
}

```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

interface IERC20 {
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function transfer(address recipient, uint256 amount)
        external
        returns (bool);
    function allowance(address owner, address spender)
        external
        view
        returns (uint256);
    function approve(address spender, uint256 amount) external returns (bool);
    function transferFrom(address sender, address recipient, uint256 amount)
        external
        returns (bool);
}

```

---
## 关注我们
[Yanbo的Twitter](https://twitter.com/YanboOfficial)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)

加入Web3Club 官方讨论群：YanboTravelAllWorld（微信号）

[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
