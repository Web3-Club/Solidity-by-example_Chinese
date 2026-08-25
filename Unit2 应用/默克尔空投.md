# 默克尔空投

对应英文原页：https://solidity-by-example.org/app/airdrop

使用默克尔树（Merkle tree）的空投合约示例。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

import {MerkleProof} from "./MerkleProof.sol";

interface IToken {
    function mint(address to, uint256 amount) external;
}

contract Airdrop {
    event Claim(address to, uint256 amount);

    IToken public immutable token;
    bytes32 public immutable root;
    mapping(bytes32 => bool) public claimed;

    constructor(address _token, bytes32 _root) {
        token = IToken(_token);
        root = _root;
    }

    function getLeafHash(address to, uint256 amount)
        public
        pure
        returns (bytes32)
    {
        return keccak256(abi.encode(to, amount));
    }

    function claim(bytes32[] memory proof, address to, uint256 amount)
        external
    {
        // 注意：(to, amount) 不能有重复
        bytes32 leaf = getLeafHash(to, amount);

        require(!claimed[leaf], "airdrop already claimed");
        require(MerkleProof.verify(proof, root, leaf), "invalid merkle proof");
        claimed[leaf] = true;

        token.mint(to, amount);

        emit Claim(to, amount);
    }
}
```

### Token

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

// ERC20 + mint + 授权
contract Token {
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
    mapping(address => bool) public authorized;

    constructor(string memory _name, string memory _symbol, uint8 _decimals) {
        name = _name;
        symbol = _symbol;
        decimals = _decimals;
        authorized[msg.sender] = true;
    }

    function setAuthorized(address addr, bool auth) external {
        require(authorized[msg.sender], "not authorized");
        authorized[addr] = auth;
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

    function mint(address to, uint256 amount) external {
        require(authorized[msg.sender], "not authorized");
        _mint(to, amount);
    }
}
```

### 从 OpenZeppelin 复制的库

```solidity
// SPDX-License-Identifier: MIT
// OpenZeppelin Contracts

pragma solidity ^0.8.20;

import {Hashes} from "./Hashes.sol";

library MerkleProof {
    function verify(bytes32[] memory proof, bytes32 root, bytes32 leaf)
        internal
        pure
        returns (bool)
    {
        return processProof(proof, leaf) == root;
    }

    function processProof(bytes32[] memory proof, bytes32 leaf)
        internal
        pure
        returns (bytes32)
    {
        bytes32 computedHash = leaf;
        for (uint256 i = 0; i < proof.length; i++) {
            computedHash = Hashes.commutativeKeccak256(computedHash, proof[i]);
        }
        return computedHash;
    }
}
```

```solidity
// SPDX-License-Identifier: MIT
// OpenZeppelin Contracts

pragma solidity ^0.8.0;

library Hashes {
    function commutativeKeccak256(bytes32 a, bytes32 b)
        internal
        pure
        returns (bytes32)
    {
        return a < b ? _efficientKeccak256(a, b) : _efficientKeccak256(b, a);
    }

    function _efficientKeccak256(bytes32 a, bytes32 b)
        private
        pure
        returns (bytes32 value)
    {
        /// @solidity memory-safe-assembly
        assembly {
            mstore(0x00, a)
            mstore(0x20, b)
            value := keccak256(0x00, 0x40)
        }
    }
}
```

### 测试

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

library MerkleHelper {
    // 冒泡排序
    function sort(bytes32[] memory arr)
        internal
        pure
        returns (bytes32[] memory)
    {
        uint256 n = arr.length;
        for (uint256 i = 0; i < n; i++) {
            for (uint256 j = 0; j < n - 1 - i; j++) {
                if (arr[j] > arr[j + 1]) {
                    (arr[j], arr[j + 1]) = (arr[j + 1], arr[j]);
                }
            }
        }

        return arr;
    }

    function yulKeccak256(bytes32 a, bytes32 b)
        internal
        pure
        returns (bytes32 v)
    {
        assembly {
            mstore(0x00, a)
            mstore(0x20, b)
            v := keccak256(0x00, 0x40)
        }
    }

    function calcRoot(bytes32[] memory hashes)
        internal
        pure
        returns (bytes32)
    {
        uint256 n = hashes.length;

        while (n > 1) {
            for (uint256 i = 0; i < n; i += 2) {
                bytes32 left = hashes[i];
                bytes32 right = hashes[i + 1 < n ? i + 1 : i];
                (left, right) = left <= right ? (left, right) : (right, left);
                hashes[i >> 1] = yulKeccak256(left, right);
            }
            n = (n + (n & 1)) >> 1;
        }

        return hashes[0];
    }

    function getProof(bytes32[] memory hashes, uint256 index)
        internal
        pure
        returns (bytes32[] memory)
    {
        bytes32[] memory proof = new bytes32[](0);
        uint256 len = 0;

        uint256 n = hashes.length;
        uint256 k = index;

        while (n > 1) {
            // 获取本层证明
            uint256 j = k & 1 == 1 ? k - 1 : (k + 1 < n ? k + 1 : k);
            bytes32 h = hashes[j];

            // proof.push(h)
            assembly {
                len := add(len, 1)
                let pos := add(proof, shl(5, len))
                mstore(pos, h)
                mstore(proof, len)
                mstore(0x40, add(pos, 0x20))
            }

            k >>= 1;

            // 计算下一层哈希
            for (uint256 i = 0; i < n; i += 2) {
                bytes32 left = hashes[i];
                bytes32 right = hashes[i + 1 < n ? i + 1 : i];
                (left, right) = left <= right ? (left, right) : (right, left);
                hashes[i >> 1] = yulKeccak256(left, right);
            }
            n = (n + (n & 1)) >> 1;
        }

        return proof;
    }

    function verify(bytes32[] memory proof, bytes32 root, bytes32 leaf)
        internal
        pure
        returns (bool)
    {
        bytes32 h = leaf;

        for (uint256 i = 0; i < proof.length; i++) {
            (bytes32 left, bytes32 right) =
                h <= proof[i] ? (h, proof[i]) : (proof[i], h);
            h = yulKeccak256(left, right);
        }

        return h == root;
    }
}
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

import {Test, console2} from "forge-std/Test.sol";
import {MerkleHelper} from "./MerkleHelper.sol";
import {Airdrop} from "../../../src/app/airdrop/Airdrop.sol";
import {Token} from "../../../src/app/airdrop/Token.sol";

contract AirdropTest is Test {
    Token private token;
    Airdrop private airdrop;

    struct Reward {
        address to;
        uint256 amount;
    }

    Reward[] private rewards;
    bytes32[] private hashes;
    mapping(bytes32 => Reward) private hashToReward;

    uint256 constant N = 100;

    function setUp() public {
        token = new Token("test", "TEST", 18);

        // 初始化用户和空投数量
        for (uint256 i = 0; i < N; i++) {
            rewards.push(
                Reward({to: address(uint160(i)), amount: (i + 1) * 100})
            );
            hashes.push(keccak256(abi.encode(rewards[i].to, rewards[i].amount)));
            hashToReward[hashes[i]] = rewards[i];
        }

        hashes = MerkleHelper.sort(hashes);

        bytes32 root = MerkleHelper.calcRoot(hashes);

        airdrop = new Airdrop(address(token), root);

        token.setAuthorized(address(airdrop), true);
    }

    function test_valid_proof() public {
        for (uint256 i = 0; i < N; i++) {
            bytes32 h = hashes[i];
            Reward memory reward = hashToReward[h];
            bytes32[] memory proof = MerkleHelper.getProof(hashes, i);

            airdrop.claim(proof, reward.to, reward.amount);
            assertEq(token.balanceOf(reward.to), reward.amount);
        }
    }
}
```

---
## 关注我们
[Yanbo的Twitter](https://x.com/Yanbo2004)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)


[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
