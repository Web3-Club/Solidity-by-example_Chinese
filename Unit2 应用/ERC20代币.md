# ERC20

遵循 <a href="https://eips.ethereum.org/EIPS/eip-20" target="__blank">ERC20 标准</a> 的任何合约都是 ERC20 代币。

ERC20 代币提供以下功能：
- 转移代币
- 允许他人代表代币持有者转移代币

## ERC20 接口

```solidity
// SPDX-License-Identifier: MIT  // SPDX 许可证标识符：MIT 许可证
pragma solidity ^0.8.26;  // 版本指令，要求使用 Solidity 0.8.26 及以上版本

// IERC20 接口：定义 ERC20 代币的标准函数签名
interface IERC20 {  // interface 关键字定义接口
    function totalSupply() external view returns (uint256);  // 返回总供应量
    function balanceOf(address account) external view returns (uint256);  // 查询某地址的余额
    function transfer(address recipient, uint256 amount)  // 转移代币
        external
        returns (bool);  // 返回是否成功
    function allowance(address owner, address spender)  // 查询授权额度
        external
        view
        returns (uint256);
    function approve(address spender, uint256 amount) external returns (bool);  // 授权某人代为转账的额度
    function transferFrom(address sender, address recipient, uint256 amount)  // 代表发送者转账
        external
        returns (bool);
}
```

## ERC20 代币合约

```solidity
// SPDX-License-Identifier: MIT  // SPDX 许可证标识符：MIT 许可证
pragma solidity ^0.8.26;  // 版本指令，要求使用 Solidity 0.8.26 及以上版本

import "./IERC20.sol";  // 导入 IERC20 接口

// ERC20 合约：实现 ERC20 标准的代币合约
contract ERC20 is IERC20 {  // 继承 IERC20 接口
    // Transfer 事件：转账时触发
    event Transfer(address indexed from, address indexed to, uint256 value);
    // Approval 事件：授权时触发
    event Approval(
        address indexed owner, address indexed spender, uint256 value
    );

    uint256 public totalSupply;  // 代币总供应量
    mapping(address => uint256) public balanceOf;  // 地址到余额的映射
    // allowance 嵌套映射：owner 授权 spender 可使用的代币数量
    mapping(address => mapping(address => uint256)) public allowance;
    string public name;     // 代币名称
    string public symbol;   // 代币符号
    uint8 public decimals;  // 小数位数

    // 构造函数：初始化代币的名称、符号和小数位数
    constructor(string memory _name, string memory _symbol, uint8 _decimals) {
        name = _name;          // 设置代币名称
        symbol = _symbol;      // 设置代币符号
        decimals = _decimals;  // 设置小数位数
    }

    // transfer 函数：将代币从调用者转移到接收者
    function transfer(address recipient, uint256 amount)
        external
        returns (bool)  // 返回是否成功
    {
        balanceOf[msg.sender] -= amount;  // 减少发送者的余额（Solidity 0.8+ 自动检查下溢）
        balanceOf[recipient] += amount;   // 增加接收者的余额
        emit Transfer(msg.sender, recipient, amount);  // 触发 Transfer 事件
        return true;  // 返回成功
    }

    // approve 函数：授权 spender 可以从调用者账户中转移一定数量的代币
    function approve(address spender, uint256 amount) external returns (bool) {
        allowance[msg.sender][spender] = amount;  // 设置授权额度
        emit Approval(msg.sender, spender, amount);  // 触发 Approval 事件
        return true;
    }

    // transferFrom 函数：代表发送者将代币转移给接收者（需要发送者预先授权）
    function transferFrom(address sender, address recipient, uint256 amount)
        external
        returns (bool)
    {
        allowance[sender][msg.sender] -= amount;  // 减少授权额度
        balanceOf[sender] -= amount;              // 减少发送者余额
        balanceOf[recipient] += amount;           // 增加接收者余额
        emit Transfer(sender, recipient, amount); // 触发 Transfer 事件
        return true;
    }

    // _mint 内部函数：铸造新代币（内部调用，增加总供应量）
    function _mint(address to, uint256 amount) internal {  // internal 只能被合约内部或子合约调用
        balanceOf[to] += amount;   // 增加目标地址的余额
        totalSupply += amount;     // 增加总供应量
        emit Transfer(address(0), to, amount);  // 从零地址转账表示铸造
    }

    // _burn 内部函数：销毁代币（内部调用，减少总供应量）
    function _burn(address from, uint256 amount) internal {
        balanceOf[from] -= amount;  // 减少来源地址的余额
        totalSupply -= amount;      // 减少总供应量
        emit Transfer(from, address(0), amount);  // 转移到零地址表示销毁
    }

    // mint 外部函数：公开的铸造接口
    function mint(address to, uint256 amount) external {
        _mint(to, amount);  // 调用内部 _mint 函数
    }

    // burn 外部函数：公开的销毁接口
    function burn(address from, uint256 amount) external {
        _burn(from, amount);  // 调用内部 _burn 函数
    }
}
```

## 创建自己的 ERC20 代币

使用 OpenZeppelin 可以非常容易地创建自己的 ERC20 代币。以下是使用上述 ERC20 合约创建自定义代币的示例：

```solidity
// SPDX-License-Identifier: MIT  // SPDX 许可证标识符：MIT 许可证
pragma solidity ^0.8.26;  // 版本指令

import "./ERC20.sol";  // 导入自定义的 ERC20 合约

// MyToken 合约：继承 ERC20，创建自定义代币
contract MyToken is ERC20 {  // 继承 ERC20 合约
    // 构造函数：设置代币名称、符号和小数位数
    constructor(string memory name, string memory symbol, uint8 decimals)
        ERC20(name, symbol, decimals)  // 调用父合约 ERC20 的构造函数
    {
        // 为 msg.sender 铸造 100 个代币
        // 类似于：
        // 1 美元 = 100 美分
        // 1 个代币 = 1 * (10 ** decimals) 最小单位
        _mint(msg.sender, 100 * 10 ** uint256(decimals));  // 铸造 100 个代币
    }
}
```

## 代币交换合约

以下是一个 `TokenSwap` 合约，用于将一种 ERC20 代币交换为另一种。它通过调用 `transferFrom` 来完成交换。

在 `TokenSwap` 调用 `transferFrom` 之前，发送者必须：
- 余额中有足够的代币
- 通过调用 `approve` 允许 `TokenSwap` 提取 `amount` 数量的代币

```solidity
// SPDX-License-Identifier: MIT  // SPDX 许可证标识符：MIT 许可证
pragma solidity ^0.8.26;  // 版本指令

import "./IERC20.sol";  // 导入 IERC20 接口

/*
代币交换流程：

1. Alice 拥有 100 枚 AliceCoin（一种 ERC20 代币）
2. Bob 拥有 100 枚 BobCoin（也是 ERC20 代币）
3. Alice 和 Bob 希望用 10 枚 AliceCoin 交换 20 枚 BobCoin
4. Alice 或 Bob 部署 TokenSwap 合约
5. Alice 调用 approve 授权 TokenSwap 从 AliceCoin 中提取 10 枚代币
6. Bob 调用 approve 授权 TokenSwap 从 BobCoin 中提取 20 枚代币
7. Alice 或 Bob 调用 TokenSwap.swap() 执行交换
8. Alice 和 Bob 成功完成代币互换
*/

// TokenSwap 合约：原子化地交换两种 ERC20 代币
contract TokenSwap {  // 定义 TokenSwap 合约
    IERC20 public token1;    // 第一种代币的合约引用
    address public owner1;   // 第一种代币的拥有者
    uint256 public amount1;  // 第一种代币的交换数量
    IERC20 public token2;    // 第二种代币的合约引用
    address public owner2;   // 第二种代币的拥有者
    uint256 public amount2;  // 第二种代币的交换数量

    // 构造函数：设置交换双方的代币、拥有者和数量
    constructor(
        address _token1,
        address _owner1,
        uint256 _amount1,
        address _token2,
        address _owner2,
        uint256 _amount2
    ) {
        token1 = IERC20(_token1);  // 将地址转换为 IERC20 接口类型
        owner1 = _owner1;           // 设置代币1的拥有者
        amount1 = _amount1;         // 设置代币1的数量
        token2 = IERC20(_token2);   // 将地址转换为 IERC20 接口类型
        owner2 = _owner2;           // 设置代币2的拥有者
        amount2 = _amount2;         // 设置代币2的数量
    }

    // swap 函数：执行原子交换
    function swap() public {  // 任意一方都可以调用
        require(msg.sender == owner1 || msg.sender == owner2, "Not authorized");  // 只有交换双方可以调用
        require(
            token1.allowance(owner1, address(this)) >= amount1,  // 检查代币1的授权额度是否足够
            "Token 1 allowance too low"
        );
        require(
            token2.allowance(owner2, address(this)) >= amount2,  // 检查代币2的授权额度是否足够
            "Token 2 allowance too low"
        );

        // 将代币1从 owner1 转给 owner2
        _safeTransferFrom(token1, owner1, owner2, amount1);  // Alice → Bob
        // 将代币2从 owner2 转给 owner1
        _safeTransferFrom(token2, owner2, owner1, amount2);  // Bob → Alice
    }

    // _safeTransferFrom 私有函数：安全地执行 transferFrom，并检查返回值
    function _safeTransferFrom(
        IERC20 token,        // 代币合约
        address sender,      // 发送者
        address recipient,   // 接收者
        uint256 amount       // 数量
    ) private {
        bool sent = token.transferFrom(sender, recipient, amount);  // 执行转账
        require(sent, "Token transfer failed");  // 检查转账是否成功
    }
}
```
