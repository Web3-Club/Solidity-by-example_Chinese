# Vault

对应英文原页：https://solidity-by-example.org/defi/vault

DeFi 协议中常用金库（vault）合约的简单示例。

主网上的大多数金库更复杂。这里我们聚焦于存款时铸造份额、以及取款时代币数量的计算。

### 合约如何工作

1. 用户存款时会铸造一定数量的份额（shares）。
2. DeFi 协议会用用户的存款产生收益（以某种方式）。
3. 用户销毁份额以提取其代币 + 收益。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

contract Vault {
    IERC20 public immutable token;

    uint256 public totalSupply;
    mapping(address => uint256) public balanceOf;

    constructor(address _token) {
        token = IERC20(_token);
    }

    function _mint(address _to, uint256 _shares) private {
        totalSupply += _shares;
        balanceOf[_to] += _shares;
    }

    function _burn(address _from, uint256 _shares) private {
        totalSupply -= _shares;
        balanceOf[_from] -= _shares;
    }

    function deposit(uint256 _amount) external {
        /*
        a = 数量
        B = 存款前的代币余额
        T = 总供应量
        s = 要铸造的份额

        (T + s) / T = (a + B) / B 

        s = aT / B
        */
        uint256 shares;
        if (totalSupply == 0) {
            shares = _amount;
        } else {
            shares = (_amount * totalSupply) / token.balanceOf(address(this));
        }

        _mint(msg.sender, shares);
        token.transferFrom(msg.sender, address(this), _amount);
    }

    function withdraw(uint256 _shares) external {
        /*
        a = 数量
        B = 取款前的代币余额
        T = 总供应量
        s = 要销毁的份额

        (T - s) / T = (B - a) / B 

        a = sB / T
        */
        uint256 amount =
            (_shares * token.balanceOf(address(this))) / totalSupply;
        _burn(msg.sender, _shares);
        token.transfer(msg.sender, amount);
    }
}

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

    event Transfer(address indexed from, address indexed to, uint256 amount);
    event Approval(
        address indexed owner, address indexed spender, uint256 amount
    );
}

```

---
## 关注我们
[Yanbo的Twitter](https://twitter.com/YanboOfficial)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)

加入Web3Club 官方讨论群：YanboTravelAllWorld（微信号）

[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
