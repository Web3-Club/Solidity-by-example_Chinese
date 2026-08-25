# 无 Gas 代币转账

对应英文原页：https://solidity-by-example.org/app/gasless-token-transfer

使用元交易（Meta transaction）实现的无 Gas ERC20 代币转账

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

interface IERC20Permit {
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
    function permit(
        address owner,
        address spender,
        uint256 value,
        uint256 deadline,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) external;
}

contract GaslessTokenTransfer {
    function send(
        address token,
        address sender,
        address receiver,
        uint256 amount,
        uint256 fee,
        uint256 deadline,
        // Permit 签名
        uint8 v,
        bytes32 r,
        bytes32 s
    ) external {
        // Permit
        IERC20Permit(token).permit(
            sender, address(this), amount + fee, deadline, v, r, s
        );
        // 将 amount 发送给接收者
        IERC20Permit(token).transferFrom(sender, receiver, amount);
        // 收取手续费 - 将 fee 发送给 msg.sender
        IERC20Permit(token).transferFrom(sender, msg.sender, fee);
    }
}
```

实现了 `permit` 的 `ERC20` 示例，复制自 solmate

```solidity
// SPDX-License-Identifier: AGPL-3.0-only
pragma solidity >=0.8.0;

/// @notice 现代且节省 Gas 的 ERC20 + EIP-2612 实现。
/// @author Solmate (https://github.com/transmissions11/solmate/blob/main/src/tokens/ERC20.sol)
/// @author 修改自 Uniswap (https://github.com/Uniswap/uniswap-v2-core/blob/master/contracts/UniswapV2ERC20.sol)
/// @dev 不要在未更新 totalSupply 的情况下手动设置余额，因为所有用户余额之和不得超过它。
abstract contract ERC20 {
    event Transfer(address indexed from, address indexed to, uint256 amount);
    event Approval(
        address indexed owner, address indexed spender, uint256 amount
    );

    string public name;
    string public symbol;
    uint8 public immutable decimals;
    uint256 public totalSupply;
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;
    uint256 internal immutable INITIAL_CHAIN_ID;
    bytes32 internal immutable INITIAL_DOMAIN_SEPARATOR;
    mapping(address => uint256) public nonces;

    constructor(string memory _name, string memory _symbol, uint8 _decimals) {
        name = _name;
        symbol = _symbol;
        decimals = _decimals;
        INITIAL_CHAIN_ID = block.chainid;
        INITIAL_DOMAIN_SEPARATOR = computeDomainSeparator();
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

        unchecked {
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
        uint256 allowed = allowance[from][msg.sender]; // 为有限额度的授权节省 Gas。
        if (allowed != type(uint256).max) {
            allowance[from][msg.sender] = allowed - amount;
        }
        balanceOf[from] -= amount;

        unchecked {
            balanceOf[to] += amount;
        }

        emit Transfer(from, to, amount);
        return true;
    }

    function permit(
        address owner,
        address spender,
        uint256 value,
        uint256 deadline,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) public virtual {
        require(deadline >= block.timestamp, "PERMIT_DEADLINE_EXPIRED");

        unchecked {
            address recoveredAddress = ecrecover(
                keccak256(
                    abi.encodePacked(
                        "\x19\x01",
                        DOMAIN_SEPARATOR(),
                        keccak256(
                            abi.encode(
                                keccak256(
                                    "Permit(address owner,address spender,uint256 value,uint256 nonce,uint256 deadline)"
                                ),
                                owner,
                                spender,
                                value,
                                nonces[owner]++,
                                deadline
                            )
                        )
                    )
                ),
                v,
                r,
                s
            );

            require(
                recoveredAddress != address(0) && recoveredAddress == owner,
                "INVALID_SIGNER"
            );

            allowance[recoveredAddress][spender] = value;
        }

        emit Approval(owner, spender, value);
    }

    function DOMAIN_SEPARATOR() public view virtual returns (bytes32) {
        return block.chainid == INITIAL_CHAIN_ID
            ? INITIAL_DOMAIN_SEPARATOR
            : computeDomainSeparator();
    }

    function computeDomainSeparator() internal view virtual returns (bytes32) {
        return keccak256(
            abi.encode(
                keccak256(
                    "EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)"
                ),
                keccak256(bytes(name)),
                keccak256("1"),
                block.chainid,
                address(this)
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

contract ERC20Permit is ERC20 {
    constructor(string memory _name, string memory _symbol, uint8 _decimals)
        ERC20(_name, _symbol, _decimals)
    {}

    function mint(address to, uint256 amount) public {
        _mint(to, amount);
    }
}
```

---
## 关注我们
[Yanbo的Twitter](https://twitter.com/YanboOfficial)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)

加入Web3Club 官方讨论群：YanboTravelAllWorld（微信号）

[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
