# ERC20

对应英文原页：https://solidity-by-example.org/app/erc20

任何遵循 [ERC20 标准](https://eips.ethereum.org/EIPS/eip-20) 的合约都是 ERC20 代币。

ERC20 代币提供以下功能

- 转移代币
- 允许他人代表代币持有者转移代币

下面是 ERC20 的接口。

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

`ERC20` 代币合约示例。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

import "./IERC20.sol";

contract ERC20 is IERC20 {
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(
        address indexed owner, address indexed spender, uint256 value
    );

    uint256 public totalSupply;
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;
    string public name;
    string public symbol;
    uint8 public decimals;

    constructor(string memory _name, string memory _symbol, uint8 _decimals) {
        name = _name;
        symbol = _symbol;
        decimals = _decimals;
    }

    function transfer(address recipient, uint256 amount)
        external
        returns (bool)
    {
        balanceOf[msg.sender] -= amount;
        balanceOf[recipient] += amount;
        emit Transfer(msg.sender, recipient, amount);
        return true;
    }

    function approve(address spender, uint256 amount) external returns (bool) {
        allowance[msg.sender][spender] = amount;
        emit Approval(msg.sender, spender, amount);
        return true;
    }

    function transferFrom(address sender, address recipient, uint256 amount)
        external
        returns (bool)
    {
        allowance[sender][msg.sender] -= amount;
        balanceOf[sender] -= amount;
        balanceOf[recipient] += amount;
        emit Transfer(sender, recipient, amount);
        return true;
    }

    function _mint(address to, uint256 amount) internal {
        balanceOf[to] += amount;
        totalSupply += amount;
        emit Transfer(address(0), to, amount);
    }

    function _burn(address from, uint256 amount) internal {
        balanceOf[from] -= amount;
        totalSupply -= amount;
        emit Transfer(from, address(0), amount);
    }

    function mint(address to, uint256 amount) external {
        _mint(to, amount);
    }

    function burn(address from, uint256 amount) external {
        _burn(from, amount);
    }
}
```

## 创建你自己的 ERC20 代币

使用 [OpenZeppelin](https://github.com/OpenZeppelin/openzeppelin-contracts) 可以非常容易地创建自己的 ERC20 代币。

下面是一个示例

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

import "./ERC20.sol";

contract MyToken is ERC20 {
    constructor(string memory name, string memory symbol, uint8 decimals)
        ERC20(name, symbol, decimals)
    {
        // 向 msg.sender 铸造 100 枚代币
        // 类似于
        // 1 美元 = 100 美分
        // 1 枚代币 = 1 * (10 ** decimals)
        _mint(msg.sender, 100 * 10 ** uint256(decimals));
    }
}
```

## 用于交换代币的合约

下面是一个示例合约 `TokenSwap`，用于将一种 ERC20 代币换成另一种。

该合约通过调用以下函数来交换代币

```solidity
transferFrom(address sender, address recipient, uint256 amount)
```

这会将 `amount` 数量的代币从 `sender` 转移到 `recipient`。

为了让 `transferFrom` 成功，`sender` 必须

- 余额中拥有超过 `amount` 的代币
- 通过调用 `approve` 允许 `TokenSwap` 提取 `amount` 代币

并且这必须在 `TokenSwap` 调用 `transferFrom` 之前完成

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

import "./IERC20.sol";

/*
如何交换代币

1. Alice 持有 100 枚 AliceCoin，这是一种 ERC20 代币。
2. Bob 持有 100 枚 BobCoin，这也是一种 ERC20 代币。
3. Alice 和 Bob 想用 10 枚 AliceCoin 交换 20 枚 BobCoin。
4. Alice 或 Bob 部署 TokenSwap
5. Alice 批准 TokenSwap 从 AliceCoin 中提取 10 枚代币
6. Bob 批准 TokenSwap 从 BobCoin 中提取 20 枚代币
7. Alice 或 Bob 调用 TokenSwap.swap()
8. Alice 和 Bob 成功交换了代币。
*/

contract TokenSwap {
    IERC20 public token1;
    address public owner1;
    uint256 public amount1;
    IERC20 public token2;
    address public owner2;
    uint256 public amount2;

    constructor(
        address _token1,
        address _owner1,
        uint256 _amount1,
        address _token2,
        address _owner2,
        uint256 _amount2
    ) {
        token1 = IERC20(_token1);
        owner1 = _owner1;
        amount1 = _amount1;
        token2 = IERC20(_token2);
        owner2 = _owner2;
        amount2 = _amount2;
    }

    function swap() public {
        require(msg.sender == owner1 || msg.sender == owner2, "Not authorized");
        require(
            token1.allowance(owner1, address(this)) >= amount1,
            "Token 1 allowance too low"
        );
        require(
            token2.allowance(owner2, address(this)) >= amount2,
            "Token 2 allowance too low"
        );

        _safeTransferFrom(token1, owner1, owner2, amount1);
        _safeTransferFrom(token2, owner2, owner1, amount2);
    }

    function _safeTransferFrom(
        IERC20 token,
        address sender,
        address recipient,
        uint256 amount
    ) private {
        bool sent = token.transferFrom(sender, recipient, amount);
        require(sent, "Token transfer failed");
    }
}
```

---
## 关注我们
[Yanbo的Twitter](https://x.com/Yanbo2004)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)


[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
