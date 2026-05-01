# DAI 代理合约

通过 DSProxy 代理与 MakerDAO 的 CDP（抵押债仓）智能合约交互，实现存入 ETH 抵押品借出 DAI 的流程。

```solidity
// SPDX-License-Identifier: MIT  // SPDX 许可证标识符：MIT 许可证
pragma solidity ^0.8.26;  // 版本指令，要求使用 Solidity 0.8.26 及以上版本

// MakerDAO 主网地址常量
address constant DAI = 0x6B175474E89094C44Da98b954EedeAC495271d0F;       // DAI 代币
address constant PROXY_REGISTRY = 0x4678f0a6958e4D2Bc4F1BAF7Bc52E8F3564f3fE4; // DSProxy 工厂
address constant PROXY_ACTIONS = 0x82ecD135Dce65Fbc6DbdD0e4237E0AF93FFD5038;   // 代理操作合约
address constant CDP_MANAGER = 0x5ef30b9986345249bc32d8928B7ee64DE9435E39;     // CDP 管理器
address constant JUG = 0x19c0976f590D67707E62397C87829d896Dc0f1F1;            // 稳定费累积器
address constant JOIN_ETH_C = 0xF04a5cC80B1E94C69B48f5ee68a08CD2F09A7c3E;    // ETH-C 抵押品接入
address constant JOIN_DAI = 0x9759A6Ac90977b93B58547b4A71c78317f391A28;       // DAI 接入

bytes32 constant ETH_C = 0x4554482d43000000000000000000000000000000000000000000000000000000; // "ETH-C"

// DaiProxy 合约：通过 DSProxy 与 MakerDAO 交互
contract DaiProxy {  // 定义 DaiProxy 合约
    IERC20 private constant dai = IERC20(DAI);
    address public immutable proxy;   // 用户的 DSProxy 地址
    uint256 public immutable cdpId;   // CDP（金库）ID

    constructor() {
        // 1. 创建 DSProxy
        proxy = IDssProxyRegistry(PROXY_REGISTRY).build();
        // 2. 通过 DSProxy 打开一个 ETH-C CDP
        bytes32 res = IDssProxy(proxy).execute(
            PROXY_ACTIONS,
            abi.encodeCall(IDssProxyActions.open, (CDP_MANAGER, ETH_C, proxy))
        );
        cdpId = uint256(res);  // 返回的 CDP ID
    }

    receive() external payable {}  // 接收 ETH

    // lockEth 函数：将 ETH 存入 CDP 作为抵押品
    function lockEth() external payable {
        IDssProxy(proxy).execute{value: msg.value}(
            PROXY_ACTIONS,
            abi.encodeCall(
                IDssProxyActions.lockETH, (CDP_MANAGER, JOIN_ETH_C, cdpId)
            )
        );
    }

    // borrow 函数：从 CDP 借出 DAI
    function borrow(uint256 daiAmount) external {
        IDssProxy(proxy).execute(
            PROXY_ACTIONS,
            abi.encodeCall(
                IDssProxyActions.draw,
                (CDP_MANAGER, JUG, JOIN_DAI, cdpId, daiAmount)
            )
        );
    }

    // repay 函数：偿还部分 DAI 借款
    function repay(uint256 daiAmount) external {
        dai.approve(proxy, daiAmount);  // 需先授权 DSProxy
        IDssProxy(proxy).execute(
            PROXY_ACTIONS,
            abi.encodeCall(
                IDssProxyActions.wipe, (CDP_MANAGER, JOIN_DAI, cdpId, daiAmount)
            )
        );
    }

    // repayAll 函数：偿还全部 DAI 借款
    function repayAll() external {
        dai.approve(proxy, type(uint256).max);  // 无限授权
        IDssProxy(proxy).execute(
            PROXY_ACTIONS,
            abi.encodeCall(
                IDssProxyActions.wipeAll, (CDP_MANAGER, JOIN_DAI, cdpId)
            )
        );
    }

    // unlockEth 函数：解锁抵押的 ETH
    function unlockEth(uint256 ethAmount) external {
        IDssProxy(proxy).execute(
            PROXY_ACTIONS,
            abi.encodeCall(
                IDssProxyActions.freeETH,
                (CDP_MANAGER, JOIN_ETH_C, cdpId, ethAmount)
            )
        );
    }
}

// MakerDAO 接口定义
interface IDssProxyRegistry {
    function build() external returns (address proxy);
}

interface IDssProxy {
    function execute(address target, bytes memory data)
        external payable returns (bytes32 res);
}

interface IDssProxyActions {
    function open(address cdpManager, bytes32 ilk, address usr) external returns (uint256 cdpId);
    function lockETH(address cdpManager, address ethJoin, uint256 cdpId) external payable;
    function draw(address cdpManager, address jug, address daiJoin, uint256 cdpId, uint256 daiAmount) external;
    function wipe(address cdpManager, address daiJoin, uint256 cdpId, uint256 daiAmount) external;
    function wipeAll(address cdpManager, address daiJoin, uint256 cdpId) external;
    function freeETH(address cdpManager, address ethJoin, uint256 cdpId, uint256 collateralAmount) external;
}

interface IERC20 { /* 标准 ERC20 接口 */ }
```

## 工作流程

1. **创建 CDP**：部署 `DaiProxy` → 自动创建 DSProxy → 打开 ETH-C CDP
2. **存入抵押品**：`lockEth()` 将 ETH 存入 CDP
3. **借出 DAI**：`borrow()` 从 CDP 借出 DAI
4. **偿还 DAI**：`repay()` / `repayAll()` 偿还借款
5. **解锁抵押品**：`unlockEth()` 取回抵押的 ETH
