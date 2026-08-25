# DAI Proxy

对应英文原页：https://solidity-by-example.org/defi/dai-proxy

### 示例

使用 `DssProxy` 锁定 ETH 抵押品、借出 DAI、偿还 DAI 并解锁 ETH 的示例。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

address constant DAI = 0x6B175474E89094C44Da98b954EedeAC495271d0F;
address constant PROXY_REGISTRY = 0x4678f0a6958e4D2Bc4F1BAF7Bc52E8F3564f3fE4;
address constant PROXY_ACTIONS = 0x82ecD135Dce65Fbc6DbdD0e4237E0AF93FFD5038;
address constant CDP_MANAGER = 0x5ef30b9986345249bc32d8928B7ee64DE9435E39;
address constant JUG = 0x19c0976f590D67707E62397C87829d896Dc0f1F1;
address constant JOIN_ETH_C = 0xF04a5cC80B1E94C69B48f5ee68a08CD2F09A7c3E;
address constant JOIN_DAI = 0x9759A6Ac90977b93B58547b4A71c78317f391A28;

bytes32 constant ETH_C =
    0x4554482d43000000000000000000000000000000000000000000000000000000;

contract DaiProxy {
    IERC20 private constant dai = IERC20(DAI);
    address public immutable proxy;
    uint256 public immutable cdpId;

    constructor() {
        proxy = IDssProxyRegistry(PROXY_REGISTRY).build();
        bytes32 res = IDssProxy(proxy).execute(
            PROXY_ACTIONS,
            abi.encodeCall(IDssProxyActions.open, (CDP_MANAGER, ETH_C, proxy))
        );
        cdpId = uint256(res);
    }

    receive() external payable {}

    function lockEth() external payable {
        IDssProxy(proxy).execute{value: msg.value}(
            PROXY_ACTIONS,
            abi.encodeCall(
                IDssProxyActions.lockETH, (CDP_MANAGER, JOIN_ETH_C, cdpId)
            )
        );
    }

    function borrow(uint256 daiAmount) external {
        IDssProxy(proxy).execute(
            PROXY_ACTIONS,
            abi.encodeCall(
                IDssProxyActions.draw,
                (CDP_MANAGER, JUG, JOIN_DAI, cdpId, daiAmount)
            )
        );
    }

    function repay(uint256 daiAmount) external {
        dai.approve(proxy, daiAmount);
        IDssProxy(proxy).execute(
            PROXY_ACTIONS,
            abi.encodeCall(
                IDssProxyActions.wipe, (CDP_MANAGER, JOIN_DAI, cdpId, daiAmount)
            )
        );
    }

    function repayAll() external {
        dai.approve(proxy, type(uint256).max);
        IDssProxy(proxy).execute(
            PROXY_ACTIONS,
            abi.encodeCall(
                IDssProxyActions.wipeAll, (CDP_MANAGER, JOIN_DAI, cdpId)
            )
        );
    }

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

interface IDssProxyRegistry {
    function build() external returns (address proxy);
}

interface IDssProxy {
    function execute(address target, bytes memory data)
        external
        payable
        returns (bytes32 res);
}

interface IDssProxyActions {
    function open(address cdpManager, bytes32 ilk, address usr)
        external
        returns (uint256 cdpId);
    function lockETH(address cdpManager, address ethJoin, uint256 cdpId)
        external
        payable;
    function draw(
        address cdpManager,
        address jug,
        address daiJoin,
        uint256 cdpId,
        uint256 daiAmount
    ) external;
    function wipe(
        address cdpManager,
        address daiJoin,
        uint256 cdpId,
        uint256 daiAmount
    ) external;
    function wipeAll(address cdpManager, address daiJoin, uint256 cdpId)
        external;
    function freeETH(
        address cdpManager,
        address ethJoin,
        uint256 cdpId,
        uint256 collateralAmount
    ) external;
}

interface IERC20 {
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function allowance(address owner, address spender)
        external
        view
        returns (uint256);
    function approve(address spender, uint256 amount) external returns (bool);
    function transfer(address dst, uint256 amount) external returns (bool);
    function transferFrom(address src, address dst, uint256 amount)
        external
        returns (bool);
}

```

### 测试

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

import {Test, console2} from "forge-std/Test.sol";
import "../../../src/defi/dai-proxy/DaiProxy.sol";

address constant VAT = 0x35D1b3F3D7966A1DFe207aa4514C12a259A0492B;

uint256 constant WAD = 1e18;
uint256 constant RAY = 1e27;
uint256 constant RAD = 1e45;

uint256 constant ETH_AMOUNT = 100 * 1e18;
uint256 constant DAI_AMOUNT = 10000 * 1e18;

contract DaiProxyTest is Test {
    IERC20 private constant dai = IERC20(DAI);
    ICdpManager private constant cdpManager = ICdpManager(CDP_MANAGER);
    IVat private constant vat = IVat(VAT);
    DaiProxy private proxy;

    function setUp() public {
        proxy = new DaiProxy();

        // 检查最低借款额
        IVat.Ilk memory ilk = vat.ilks(ETH_C);
        assertGe(DAI_AMOUNT * RAY, ilk.dust, "DAI borrow amount < dust");

        // 利率累加器
        console2.log("ilk.rate", ilk.rate);
    }

    function print(address urnAddr) private {
        IVat.Urn memory urn = vat.urns(ETH_C, urnAddr);
        console2.log("--------------------");
        console2.log("vault collateral [wad]", urn.ink);
        console2.log("vault debt       [wad]", urn.art);
        console2.log("DAI in proxy     [wad]", dai.balanceOf(address(proxy)));
        console2.log("ETH in proxy     [wad]", address(proxy).balance);
    }

    function test_proxy() public {
        uint256 cdpId = proxy.cdpId();
        address urnAddr = cdpManager.urns(cdpId);

        console2.log("Before lock ETH");
        print(urnAddr);

        proxy.lockEth{value: ETH_AMOUNT}();
        console2.log("");
        console2.log("After lock ETH");
        print(urnAddr);

        proxy.borrow(DAI_AMOUNT);
        console2.log("");
        console2.log("After borrow DAI");
        print(urnAddr);

        proxy.repay(DAI_AMOUNT / 2);
        console2.log("");
        console2.log("After partial repay DAI");
        print(urnAddr);

        proxy.repayAll();
        console2.log("");
        console2.log("After repay all DAI");
        print(urnAddr);

        proxy.unlockEth(ETH_AMOUNT);
        console2.log("");
        console2.log("After unlock ETH");
        print(urnAddr);
    }
}

interface IVat {
    // 抵押品类型
    struct Ilk {
        uint256 Art; // 标准化债务总量      [wad]
        uint256 rate; // 累计利率           [ray]
        uint256 spot; // 含安全边际的价格   [ray]
        uint256 line; // 债务上限           [rad]
        uint256 dust; // Urn 债务下限       [rad]
    }

    // 金库
    struct Urn {
        uint256 ink; // 锁定的抵押品  [wad]
        uint256 art; // 标准化债务    [wad]
    }

    function ilks(bytes32 ilk) external view returns (Ilk memory);
    function urns(bytes32 ilk, address user)
        external
        view
        returns (Urn memory);
}

interface ICdpManager {
    function urns(uint256 cdpId) external view returns (address urn);
}

```

---
## 关注我们
[Yanbo的Twitter](https://twitter.com/YanboOfficial)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)

加入Web3Club 官方讨论群：YanboTravelAllWorld（微信号）

[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
