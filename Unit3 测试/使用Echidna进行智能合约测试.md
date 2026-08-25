# 使用 Echidna 测试智能合约

对应英文原页：https://solidity-by-example.org/tests/echidna

使用 [Echidna](https://github.com/crytic/echidna) 进行模糊测试的示例。

1. 将 solidity 合约保存为 `TestEchidna.sol`
2. 在存储合约的文件夹中执行以下命令。

```shell
docker run -it --rm -v $PWD:/code trailofbits/eth-security-toolbox
```

在 docker 内部，你的代码会存放在根目录下的 `/code`。

3. 参见下面的注释并执行 `echidna` 命令。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

/*
echidna TestEchidna.sol --contract TestCounter
*/
contract Counter {
    uint256 public count;

    function inc() external {
        count += 1;
    }

    function dec() external {
        count -= 1;
    }
}

contract TestCounter is Counter {
    function echidna_test_true() public view returns (bool) {
        return true;
    }

    function echidna_test_false() public view returns (bool) {
        return false;
    }

    function echidna_test_count() public view returns (bool) {
        // 这里我们测试 Counter.count 应始终 <= 5。
        // 测试会失败。Echidna 足够聪明，会调用 Counter.inc()
        // 超过 5 次。
        return count <= 5;
    }
}

/*
echidna TestEchidna.sol --contract TestAssert --test-mode assertion
*/
contract TestAssert {
    function test_assert(uint256 _i) external {
        assert(_i < 10);
    }

    // 更复杂的示例
    function abs(uint256 x, uint256 y) private pure returns (uint256) {
        if (x >= y) {
            return x - y;
        }
        return y - x;
    }

    function test_abs(uint256 x, uint256 y) external {
        uint256 z = abs(x, y);
        if (x >= y) {
            assert(z <= x);
        } else {
            assert(z <= y);
        }
    }
}

```

### 测试时间和调用者

Echidna 可以对时间戳进行模糊测试。时间戳范围在配置中设置。默认是 7 天。

合约调用者也可以在配置中设置。默认账户为

- `0x10000`
- `0x20000`
- `0x30000`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

/*
docker run -it --rm -v $PWD:/code trailofbits/eth-security-toolbox
echidna EchidnaTestTimeAndCaller.sol --contract EchidnaTestTimeAndCaller
*/
contract EchidnaTestTimeAndCaller {
    bool private pass = true;
    uint256 private createdAt = block.timestamp;

    /*
    如果 Echidna 能调用 setFail()，测试会失败
    否则测试会通过
    */
    function echidna_test_pass() public view returns (bool) {
        return pass;
    }

    function setFail() external {
        /*
        如果 delay <= 最大区块延迟，Echidna 可以调用此函数
        否则 Echidna 将无法调用此函数。
        可以通过配置文件指定来延长最大区块延迟。
        */
        uint256 delay = 7 days;
        require(block.timestamp >= createdAt + delay);
        pass = false;
    }

    // 默认发送者
    // 更改这些地址以查看测试失败
    address[3] private senders =
        [address(0x10000), address(0x20000), address(0x30000)];

    address private sender = msg.sender;

    // 将 _sender 作为输入，并要求 msg.sender == _sender
    // 以便在反例中看到 _sender
    function setSender(address _sender) external {
        require(_sender == msg.sender);
        sender = msg.sender;
    }

    // 检查默认发送者。发送者应为 3 个默认账户之一。
    function echidna_test_sender() public view returns (bool) {
        for (uint256 i; i < 3; i++) {
            if (sender == senders[i]) {
                return true;
            }
        }
        return false;
    }
}

```

---
## 关注我们
[Yanbo的Twitter](https://twitter.com/YanboOfficial)｜[Web3Club的Twitter](https://twitter.com/Web3ClubCN)

加入Web3Club 官方讨论群：YanboTravelAllWorld（微信号）

[加入我们](https://github.com/Web3-Club/Intro./blob/main/Join%20club.md)
