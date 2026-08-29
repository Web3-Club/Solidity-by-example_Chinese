# 双向支付渠道

对应英文原页：https://solidity-by-example.org/app/bi-directional-payment-channel

双向支付渠道允许参与者 `Alice` 和 `Bob` 反复在链下转移以太币。

付款可以双向进行，`Alice` 付给 `Bob`，`Bob` 也可以付给 `Alice`。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

import "./ECDSA.sol";

/*
打开通道
1. Alice 和 Bob 向一个多签钱包注资
2. 预计算支付通道地址
3. Alice 和 Bob 交换初始余额的签名
4. Alice 和 Bob 创建一笔交易，可从多签钱包部署支付通道

更新通道余额
1. 重复打开通道中的步骤 1 - 3
2. 从多签钱包创建一笔交易，该交易会
   - 删除本来会部署旧支付通道的那笔交易
   - 然后创建一笔可以用新余额部署支付通道的交易

Alice 和 Bob 就最终余额达成一致时关闭通道
1. 从多签钱包创建一笔交易，该交易会
   - 向 Alice 和 Bob 打款
   - 然后删除本来会创建支付通道的那笔交易

Alice 和 Bob 对最终余额达不成一致时关闭通道
1. 从多签钱包部署支付通道
2. 调用 challengeExit() 启动关闭通道流程
3. 通道过期后，Alice 和 Bob 可以提取资金
*/

contract BiDirectionalPaymentChannel {
    using ECDSA for bytes32;

    event ChallengeExit(address indexed sender, uint256 nonce);
    event Withdraw(address indexed to, uint256 amount);

    address payable[2] public users;
    mapping(address => bool) public isUser;

    mapping(address => uint256) public balances;

    uint256 public challengePeriod;
    uint256 public expiresAt;
    uint256 public nonce;

    modifier checkBalances(uint256[2] memory _balances) {
        require(
            address(this).balance >= _balances[0] + _balances[1],
            "balance of contract must be >= to the total balance of users"
        );
        _;
    }

    // 注意：从多签钱包存款
    constructor(
        address payable[2] memory _users,
        uint256[2] memory _balances,
        uint256 _expiresAt,
        uint256 _challengePeriod
    ) payable checkBalances(_balances) {
        require(_expiresAt > block.timestamp, "Expiration must be > now");
        require(_challengePeriod > 0, "Challenge period must be > 0");

        for (uint256 i = 0; i < _users.length; i++) {
            address payable user = _users[i];

            require(!isUser[user], "user must be unique");
            users[i] = user;
            isUser[user] = true;

            balances[user] = _balances[i];
        }

        expiresAt = _expiresAt;
        challengePeriod = _challengePeriod;
    }

    function verify(
        bytes[2] memory _signatures,
        address _contract,
        address[2] memory _signers,
        uint256[2] memory _balances,
        uint256 _nonce
    ) public pure returns (bool) {
        for (uint256 i = 0; i < _signatures.length; i++) {
            /*
            注意：用本合约地址签名，防止在其他合约上发生重放攻击
            */
            bool valid = _signers[i]
                == keccak256(abi.encodePacked(_contract, _balances, _nonce))
                    .toEthSignedMessageHash().recover(_signatures[i]);

            if (!valid) {
                return false;
            }
        }

        return true;
    }

    modifier checkSignatures(
        bytes[2] memory _signatures,
        uint256[2] memory _balances,
        uint256 _nonce
    ) {
        // 注意：把 storage 数组复制到 memory
        address[2] memory signers;
        for (uint256 i = 0; i < users.length; i++) {
            signers[i] = users[i];
        }

        require(
            verify(_signatures, address(this), signers, _balances, _nonce),
            "Invalid signature"
        );

        _;
    }

    modifier onlyUser() {
        require(isUser[msg.sender], "Not user");
        _;
    }

    function challengeExit(
        uint256[2] memory _balances,
        uint256 _nonce,
        bytes[2] memory _signatures
    )
        public
        onlyUser
        checkSignatures(_signatures, _balances, _nonce)
        checkBalances(_balances)
    {
        require(block.timestamp < expiresAt, "Expired challenge period");
        require(_nonce > nonce, "Nonce must be greater than the current nonce");

        for (uint256 i = 0; i < _balances.length; i++) {
            balances[users[i]] = _balances[i];
        }

        nonce = _nonce;
        expiresAt = block.timestamp + challengePeriod;

        emit ChallengeExit(msg.sender, nonce);
    }

    function withdraw() public onlyUser {
        require(
            block.timestamp >= expiresAt, "Challenge period has not expired yet"
        );

        uint256 amount = balances[msg.sender];
        balances[msg.sender] = 0;

        (bool sent,) = msg.sender.call{value: amount}("");
        require(sent, "Failed to send Ether");

        emit Withdraw(msg.sender, amount);
    }
}
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

// OpenZeppelin Contracts (last updated v4.5.0) (utils/cryptography/ECDSA.sol)

library ECDSA {
    enum RecoverError {
        NoError,
        InvalidSignature,
        InvalidSignatureLength,
        InvalidSignatureS,
        InvalidSignatureV
    }

    function _throwError(RecoverError error) private pure {
        if (error == RecoverError.NoError) {
            return; // 无错误：什么都不做
        } else if (error == RecoverError.InvalidSignature) {
            revert("ECDSA: invalid signature");
        } else if (error == RecoverError.InvalidSignatureLength) {
            revert("ECDSA: invalid signature length");
        } else if (error == RecoverError.InvalidSignatureS) {
            revert("ECDSA: invalid signature 's' value");
        } else if (error == RecoverError.InvalidSignatureV) {
            revert("ECDSA: invalid signature 'v' value");
        }
    }

    function tryRecover(bytes32 hash, bytes memory signature)
        internal
        pure
        returns (address, RecoverError)
    {
        // 检查签名长度
        // - 65 字节：r,s,v 签名（标准）
        // - 64 字节：r,vs 签名（参见 https://eips.ethereum.org/EIPS/eip-2098）_自 v4.1 起可用_
        if (signature.length == 65) {
            bytes32 r;
            bytes32 s;
            uint8 v;
            // ecrecover 需要签名参数，目前只能通过汇编取得它们。
            assembly {
                r := mload(add(signature, 0x20))
                s := mload(add(signature, 0x40))
                v := byte(0, mload(add(signature, 0x60)))
            }
            return tryRecover(hash, v, r, s);
        } else if (signature.length == 64) {
            bytes32 r;
            bytes32 vs;
            // ecrecover 需要签名参数，目前只能通过汇编取得它们。
            assembly {
                r := mload(add(signature, 0x20))
                vs := mload(add(signature, 0x40))
            }
            return tryRecover(hash, r, vs);
        } else {
            return (address(0), RecoverError.InvalidSignatureLength);
        }
    }

    function recover(bytes32 hash, bytes memory signature)
        internal
        pure
        returns (address)
    {
        (address recovered, RecoverError error) = tryRecover(hash, signature);
        _throwError(error);
        return recovered;
    }

    function tryRecover(bytes32 hash, bytes32 r, bytes32 vs)
        internal
        pure
        returns (address, RecoverError)
    {
        bytes32 s = vs
            & bytes32(
                0x7fffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
            );
        uint8 v = uint8((uint256(vs) >> 255) + 27);
        return tryRecover(hash, v, r, s);
    }

    function recover(bytes32 hash, bytes32 r, bytes32 vs)
        internal
        pure
        returns (address)
    {
        (address recovered, RecoverError error) = tryRecover(hash, r, vs);
        _throwError(error);
        return recovered;
    }

    function tryRecover(bytes32 hash, uint8 v, bytes32 r, bytes32 s)
        internal
        pure
        returns (address, RecoverError)
    {
        // EIP-2 仍允许 ecrecover() 存在签名延展性。去掉这种可能，使签名唯一。
        // 以太坊黄皮书附录 F（https://ethereum.github.io/yellowpaper/paper.pdf）定义了
        // (301) 中 s 的有效范围：0 < s < secp256k1n ÷ 2 + 1，以及 (302) 中 v 的范围：v ∈ {27, 28}。
        // 当前大多数库生成的签名，其 s 值位于下半阶，因此是唯一的。
        //
        // 如果你的库生成可延展签名，例如 s 值落在上半区，请用
        // 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141 - s1 计算新的 s，
        // 并把 v 从 27 翻转到 28 或反过来。如果你的库为 v 生成 0/1 而不是 27/28，把 27 加到 v 上，
        // 也可以接受这些可延展签名。
        if (
            uint256(s)
                > 0x7FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF5D576E7357A4501DDFE92F46681B20A0
        ) {
            return (address(0), RecoverError.InvalidSignatureS);
        }
        if (v != 27 && v != 28) {
            return (address(0), RecoverError.InvalidSignatureV);
        }

        // 若签名有效（且不可延展），返回签名者地址
        address signer = ecrecover(hash, v, r, s);
        if (signer == address(0)) {
            return (address(0), RecoverError.InvalidSignature);
        }

        return (signer, RecoverError.NoError);
    }

    function recover(bytes32 hash, uint8 v, bytes32 r, bytes32 s)
        internal
        pure
        returns (address)
    {
        (address recovered, RecoverError error) = tryRecover(hash, v, r, s);
        _throwError(error);
        return recovered;
    }

    function toEthSignedMessageHash(bytes32 hash)
        internal
        pure
        returns (bytes32)
    {
        // 32 是哈希的字节长度，
        // 由上面的类型签名强制保证
        return keccak256(
            abi.encodePacked("\x19Ethereum Signed Message:\n32", hash)
        );
    }
}
```

---
## 关注我们
[Yanbo的Twitter](https://x.com/Yanbo2004)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)


[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
