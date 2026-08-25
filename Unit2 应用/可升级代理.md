# 可升级代理

对应英文原页：https://solidity-by-example.org/app/upgradeable-proxy

可升级代理合约示例。切勿用于生产环境。

本示例展示：

- 如何在调用 `fallback` 时使用 `delegatecall` 并返回数据。
- 如何把 `admin` 和 `implementation` 的地址存在特定存储槽。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

// 透明可升级代理模式

contract CounterV1 {
    uint256 public count;

    function inc() external {
        count += 1;
    }
}

contract CounterV2 {
    uint256 public count;

    function inc() external {
        count += 1;
    }

    function dec() external {
        count -= 1;
    }
}

contract BuggyProxy {
    address public implementation;
    address public admin;

    constructor() {
        admin = msg.sender;
    }

    function _delegate() private {
        (bool ok,) = implementation.delegatecall(msg.data);
        require(ok, "delegatecall failed");
    }

    fallback() external payable {
        _delegate();
    }

    receive() external payable {
        _delegate();
    }

    function upgradeTo(address _implementation) external {
        require(msg.sender == admin, "not authorized");
        implementation = _implementation;
    }
}

contract Dev {
    function selectors() external view returns (bytes4, bytes4, bytes4) {
        return (
            Proxy.admin.selector,
            Proxy.implementation.selector,
            Proxy.upgradeTo.selector
        );
    }
}

contract Proxy {
    // 所有函数 / 变量都应是 private，把所有调用转发到 fallback

    // 对未知原像减 1
    // 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc
    bytes32 private constant IMPLEMENTATION_SLOT =
        bytes32(uint256(keccak256("eip1967.proxy.implementation")) - 1);
    // 0xb53127684a568b3173ae13b9f8a6016e243e63b6e8ee1178d6a717850b5d6103
    bytes32 private constant ADMIN_SLOT =
        bytes32(uint256(keccak256("eip1967.proxy.admin")) - 1);

    constructor() {
        _setAdmin(msg.sender);
    }

    modifier ifAdmin() {
        if (msg.sender == _getAdmin()) {
            _;
        } else {
            _fallback();
        }
    }

    function _getAdmin() private view returns (address) {
        return StorageSlot.getAddressSlot(ADMIN_SLOT).value;
    }

    function _setAdmin(address _admin) private {
        require(_admin != address(0), "admin = zero address");
        StorageSlot.getAddressSlot(ADMIN_SLOT).value = _admin;
    }

    function _getImplementation() private view returns (address) {
        return StorageSlot.getAddressSlot(IMPLEMENTATION_SLOT).value;
    }

    function _setImplementation(address _implementation) private {
        require(
            _implementation.code.length > 0, "implementation is not a contract"
        );
        StorageSlot.getAddressSlot(IMPLEMENTATION_SLOT).value = _implementation;
    }

    // 管理员接口 //
    function changeAdmin(address _admin) external ifAdmin {
        _setAdmin(_admin);
    }

    // 0x3659cfe6
    function upgradeTo(address _implementation) external ifAdmin {
        _setImplementation(_implementation);
    }

    // 0xf851a440
    function admin() external ifAdmin returns (address) {
        return _getAdmin();
    }

    // 0x5c60da1b
    function implementation() external ifAdmin returns (address) {
        return _getImplementation();
    }

    // 用户接口 //
    function _delegate(address _implementation) internal virtual {
        assembly {
            // 复制 msg.data。这段内联汇编不会返回到 Solidity 代码，
            // 因此我们完全接管内存，并覆盖内存位置 0 处的 Solidity 暂存区。

            // calldatacopy(t, f, s) - 把 calldata 从位置 f 起的 s 字节复制到内存位置 t
            // calldatasize() - calldata 的字节大小
            calldatacopy(0, 0, calldatasize())

            // 调用实现合约。
            // 此时还不知道返回数据大小，所以 out 和 outsize 为 0。

            // delegatecall(g, a, in, insize, out, outsize) -
            // - 调用地址 a 的合约
            // - 输入为 mem[in…(in+insize))
            // - 提供 g 的 gas
            // - 输出区域为 mem[out…(out+outsize))
            // - 出错（例如 gas 不足）返回 0，成功返回 1
            let result :=
                delegatecall(gas(), _implementation, 0, calldatasize(), 0, 0)

            // 复制返回数据。
            // returndatacopy(t, f, s) - 把 returndata 从位置 f 起的 s 字节复制到内存位置 t
            // returndatasize() - 上次返回数据的大小
            returndatacopy(0, 0, returndatasize())

            switch result
            // delegatecall 出错时返回 0。
            case 0 {
                // revert(p, s) - 结束执行、回滚状态，返回 mem[p…(p+s))
                revert(0, returndatasize())
            }
            default {
                // return(p, s) - 结束执行，返回 mem[p…(p+s))
                return(0, returndatasize())
            }
        }
    }

    function _fallback() private {
        _delegate(_getImplementation());
    }

    fallback() external payable {
        _fallback();
    }

    receive() external payable {
        _fallback();
    }
}

contract ProxyAdmin {
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "not owner");
        _;
    }

    function getProxyAdmin(address proxy) external view returns (address) {
        (bool ok, bytes memory res) =
            proxy.staticcall(abi.encodeCall(Proxy.admin, ()));
        require(ok, "call failed");
        return abi.decode(res, (address));
    }

    function getProxyImplementation(address proxy)
        external
        view
        returns (address)
    {
        (bool ok, bytes memory res) =
            proxy.staticcall(abi.encodeCall(Proxy.implementation, ()));
        require(ok, "call failed");
        return abi.decode(res, (address));
    }

    function changeProxyAdmin(address payable proxy, address admin)
        external
        onlyOwner
    {
        Proxy(proxy).changeAdmin(admin);
    }

    function upgrade(address payable proxy, address implementation)
        external
        onlyOwner
    {
        Proxy(proxy).upgradeTo(implementation);
    }
}

library StorageSlot {
    struct AddressSlot {
        address value;
    }

    function getAddressSlot(bytes32 slot)
        internal
        pure
        returns (AddressSlot storage r)
    {
        assembly {
            r.slot := slot
        }
    }
}

contract TestSlot {
    bytes32 public constant slot = keccak256("TEST_SLOT");

    function getSlot() external view returns (address) {
        return StorageSlot.getAddressSlot(slot).value;
    }

    function writeSlot(address _addr) external {
        StorageSlot.getAddressSlot(slot).value = _addr;
    }
}
```

---
## 关注我们
[Yanbo的Twitter](https://x.com/Yanbo2004)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)


[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
