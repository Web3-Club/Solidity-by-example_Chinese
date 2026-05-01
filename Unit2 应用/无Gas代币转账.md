# 无Gas代币转账

使用元交易（Meta Transaction）实现免 Gas 的 ERC20 代币转账。通过 `permit` 签名授权，由第三方代为支付 Gas 费用。

```solidity
// SPDX-License-Identifier: MIT  // SPDX 许可证标识符：MIT 许可证
pragma solidity ^0.8.26;  // 版本指令，要求使用 Solidity 0.8.26 及以上版本

// IERC20Permit 接口：支持 permit 签名的 ERC20 接口
interface IERC20Permit {  // 定义支持 permit 的 ERC20 接口
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
    // permit 函数：通过链下签名授权 transferFrom，无需用户支付 Gas
    function permit(
        address owner,     // 代币所有者地址
        address spender,   // 被授权者地址
        uint256 value,     // 授权金额
        uint256 deadline,  // 截止时间（Unix 时间戳）
        uint8 v,           // 签名恢复标识符
        bytes32 r,         // 签名的 r 部分
        bytes32 s          // 签名的 s 部分
    ) external;
}

// GaslessTokenTransfer 合约：无 Gas 代币转账的中继合约
contract GaslessTokenTransfer {  // 定义 GaslessTokenTransfer 合约
    // send 函数：执行无 Gas 代币转账
    // 流程：
    // 1. 用户链下签名 permit 消息
    // 2. 中继者（支付 Gas 的人）调用此函数
    // 3. 合约先执行 permit 授权，再执行转账
    function send(
        address token,      // 代币合约地址
        address sender,     // 代币发送者（签名的用户）
        address receiver,   // 代币接收者
        uint256 amount,     // 转账金额
        uint256 fee,        // 给中继者的费用
        uint256 deadline,   // 签名截止时间
        // Permit 签名参数（由发送者链下生成）
        uint8 v,            // 签名 v 值
        bytes32 r,          // 签名 r 值
        bytes32 s           // 签名 s 值
    ) external {
        // 第一步：执行 Permit
        // 授权本合约从 sender 账户中转移 amount + fee 数量的代币
        IERC20Permit(token).permit(
            sender, address(this), amount + fee, deadline, v, r, s  // 通过签名完成链上授权
        );
        // 第二步：将 amount 数量的代币从 sender 转给 receiver
        IERC20Permit(token).transferFrom(sender, receiver, amount);  // 转账给接收者
        // 第三步：将 fee 数量的代币从 sender 转给 msg.sender（中继者费用）
        IERC20Permit(token).transferFrom(sender, msg.sender, fee);  // 支付中继费用
    }
}
```

## 带 Permit 的 ERC20 合约

以下是一个实现了 `permit`（EIP-2612）的现代高效 ERC20 合约，参考自 Solmate：

```solidity
// SPDX-License-Identifier: AGPL-3.0-only  // AGPL 许可证
pragma solidity >=0.8.0;  // 版本指令（为了兼容性使用了 >=）

// 参考来源：
// @notice 现代且 Gas 高效的 ERC20 + EIP-2612 实现
// @author Solmate (https://github.com/transmissions11/solmate/blob/main/src/tokens/ERC20.sol)
// @author 修改自 Uniswap (https://github.com/Uniswap/uniswap-v2-core/blob/master/contracts/UniswapV2ERC20.sol)
abstract contract ERC20 {  // 抽象合约，需要被子合约继承使用
    event Transfer(address indexed from, address indexed to, uint256 amount);
    event Approval(
        address indexed owner, address indexed spender, uint256 amount
    );

    string public name;                       // 代币名称
    string public symbol;                     // 代币符号
    uint8 public immutable decimals;          // 小数位数（不可变）
    uint256 public totalSupply;               // 总供应量
    mapping(address => uint256) public balanceOf;  // 余额映射
    mapping(address => mapping(address => uint256)) public allowance;  // 授权额度映射
    uint256 internal immutable INITIAL_CHAIN_ID;           // 部署时的链 ID
    bytes32 internal immutable INITIAL_DOMAIN_SEPARATOR;   // 初始域分隔符
    mapping(address => uint256) public nonces;  // 每个地址的 nonce（用于 permit 防重放）

    constructor(string memory _name, string memory _symbol, uint8 _decimals) {
        name = _name;
        symbol = _symbol;
        decimals = _decimals;
        INITIAL_CHAIN_ID = block.chainid;  // 记录部署时的链 ID
        INITIAL_DOMAIN_SEPARATOR = computeDomainSeparator();  // 计算域分隔符并缓存
    }

    function approve(address spender, uint256 amount)
        public
        virtual
        returns (bool)
    {
        allowance[msg.sender][spender] = amount;
        emit Approval(msg.sender, spender, amount);
        return true;
    }

    function transfer(address to, uint256 amount)
        public
        virtual
        returns (bool)
    {
        balanceOf[msg.sender] -= amount;
        unchecked {  // unchecked 节省 gas
            balanceOf[to] += amount;
        }
        emit Transfer(msg.sender, to, amount);
        return true;
    }

    function transferFrom(address from, address to, uint256 amount)
        public
        virtual
        returns (bool)
    {
        uint256 allowed = allowance[from][msg.sender];  // 获取授权额度
        if (allowed != type(uint256).max) {  // 如果不是无限授权（节省存储写入 gas）
            allowance[from][msg.sender] = allowed - amount;  // 扣除额度
        }
        balanceOf[from] -= amount;
        unchecked {
            balanceOf[to] += amount;
        }
        emit Transfer(from, to, amount);
        return true;
    }

    // permit 函数：EIP-2612 签名授权
    // 用户通过链下签名 approve 消息，避免了两次交易（先 approve 再 transferFrom）
    function permit(
        address owner,
        address spender,
        uint256 value,
        uint256 deadline,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) public virtual {
        require(deadline >= block.timestamp, "PERMIT_DEADLINE_EXPIRED");  // 检查截止时间

        unchecked {  // 节省 nonce 递增的溢出检查 gas
            // 使用 ecrecover 从签名恢复地址
            address recoveredAddress = ecrecover(
                keccak256(
                    abi.encodePacked(
                        "\x19\x01",                       // EIP-712 前缀
                        DOMAIN_SEPARATOR(),               // 域分隔符
                        keccak256(                        // Permit 类型哈希
                            abi.encode(
                                keccak256(
                                    "Permit(address owner,address spender,uint256 value,uint256 nonce,uint256 deadline)"
                                ),
                                owner,    // 所有者地址
                                spender,  // 被授权者地址
                                value,    // 授权金额
                                nonces[owner]++,  // current nonce，使用后递增
                                deadline  // 截止时间
                            )
                        )
                    )
                ),
                v,  // 签名 v
                r,  // 签名 r
                s   // 签名 s
            );

            require(
                recoveredAddress != address(0) && recoveredAddress == owner,  // 恢复的地址必须与 owner 一致
                "INVALID_SIGNER"
            );

            allowance[recoveredAddress][spender] = value;  // 设置授权额度
        }

        emit Approval(owner, spender, value);  // 触发 Approval 事件
    }

    // DOMAIN_SEPARATOR 函数：返回 EIP-712 域分隔符
    function DOMAIN_SEPARATOR() public view virtual returns (bytes32) {
        // 如果链 ID 没有变化，返回缓存的域分隔符（节省 gas）
        return block.chainid == INITIAL_CHAIN_ID
            ? INITIAL_DOMAIN_SEPARATOR
            : computeDomainSeparator();  // 链 ID 变化时重新计算
    }

    // computeDomainSeparator 内部函数：计算 EIP-712 域分隔符
    function computeDomainSeparator() internal view virtual returns (bytes32) {
        return keccak256(
            abi.encode(
                keccak256(
                    "EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)"
                ),
                keccak256(bytes(name)),  // 代币名称哈希
                keccak256("1"),          // 版本号
                block.chainid,           // 当前链 ID
                address(this)            // 本合约地址（验证合约）
            )
        );
    }

    function _mint(address to, uint256 amount) internal virtual {
        totalSupply += amount;
        unchecked {
            balanceOf[to] += amount;
        }
        emit Transfer(address(0), to, amount);
    }

    function _burn(address from, uint256 amount) internal virtual {
        balanceOf[from] -= amount;
        unchecked {
            totalSupply -= amount;
        }
        emit Transfer(from, address(0), amount);
    }
}

// ERC20Permit 合约：继承 ERC20 的可铸造代币
contract ERC20Permit is ERC20 {  // 继承 ERC20 抽象合约
    constructor(string memory _name, string memory _symbol, uint8 _decimals)
        ERC20(_name, _symbol, _decimals)  // 调用父合约构造函数
    {}

    // mint 函数：铸造新代币
    function mint(address to, uint256 amount) public {
        _mint(to, amount);  // 调用内部 _mint 函数
    }
}
```
