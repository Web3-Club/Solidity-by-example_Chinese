## ERC20 代币合约

ERC20 代币合约是以太坊上最常见的合约之一，它定义了一些标准化的接口和功能，使得不同的代币可以在以太坊上互相交换、转移等。其全称为“以太坊请求代币”（Ethereum Request for Comment 20）。

# 示例

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.4;

contract MyToken {
    string public name;
    string public symbol;
    uint8 public decimals;
    uint256 public totalSupply;

    mapping(address => uint256) private balances;
    mapping(address => mapping(address => uint256)) private allowances;

    constructor(
        string memory _name,
        string memory _symbol,
        uint8 _decimals,
        uint256 _totalSupply
    ) {
        name = _name;
        symbol = _symbol;
        decimals = _decimals;
        totalSupply = _totalSupply;

        balances[msg.sender] = totalSupply;
    }

    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(
        address indexed owner,
        address indexed spender,
        uint256 value
    );

    function balanceOf(address account) public view returns (uint256) {
        return balances[account];
    }

    function transfer(address recipient, uint256 amount)
        public
        returns (bool)
    {
        require(recipient != address(0), "MyToken: transfer to zero address");
        require(
            amount <= balances[msg.sender],
            "MyToken: transfer amount exceeds balance"
        );

        balances[msg.sender] -= amount;
        balances[recipient] += amount;

        emit Transfer(msg.sender, recipient, amount);

        return true;
    }

    function approve(address spender, uint256 amount) public returns (bool) {
        require(spender != address(0), "MyToken: approve to zero address");

        allowances[msg.sender][spender] = amount;

        emit Approval(msg.sender, spender, amount);

        return true;
    }

    function allowance(address owner, address spender)
        public
        view
        returns (uint256)
    {
        return allowances[owner][spender];
    }

    function transferFrom(
        address sender,
        address recipient,
        uint256 amount
    ) public returns (bool) {
        require(sender != address(0), "MyToken: transfer from zero address");
        require(recipient != address(0), "MyToken: transfer to zero address");
        require(
            amount <= balances[sender],
            "MyToken: transfer amount exceeds balance"
        );
        require(
            amount <= allowances[sender][msg.sender],
            "MyToken: transfer amount exceeds allowance"
        );

        balances[sender] -= amount;
        balances[recipient] += amount;
        allowances[sender][msg.sender] -= amount;

        emit Transfer(sender, recipient, amount);

        return true;
    }
}
```

该合约具有以下功能：

- name：代币名称
- symbol：代币符号
- decimals：小数点位数
- totalSupply：代币总供应量
- balanceOf：返回指定账户的余额
- transfer：将代币从一个账户转移到另一个账户
- approve：授权第三方地址可以从发送人账户转移一定数量的代币
- allowance：查询第三方地址还可以从发送人账户转移多少代币
- transferFrom：从一个账户向另一个账户转移代币，并减少授权的代币数量

在部署合约时，我们需要传入一些参数，如代币名称、代币符号、小数点位数和代币总供应量等。然后，在构造函数中，我们将所有的代币分配给合约的创建人。

在转移代币时，我们需要检查目标地址是否为零地址、发送人是否有足够的代币余额，并且需要更新相应地址的代币余额。在批准第三方地址转移代币时，我们只需将要授权的代币数量存储在映射 allowances 中即可。在进行授权后，第三方地址就可以使用 transferFrom 函数来转移指定数量的代币了