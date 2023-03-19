# Gas费

你需要支付多少以太币来进行一次交易？
你需要支付 gas 消耗量 * gas 价格 的以太币，其中：

gas 是计算单位\
gas 消耗量是交易中使用的总 gas 数量\
gas 价格是你愿意支付的每单位 gas 的以太币\
具有更高 gas 价格的交易具有更高的优先级被包含在区块中。\
未使用的 gas 将被退还。

燃气限制\
你可以花费的 gas 有两个上限\
gas 限制（你为交易设置的最大 gas 量）\
区块 gas 限制（网络设置的区块中允许的最大 gas 量）

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

contract Gas {
uint public i = 0;

  // 使用完你发送的所有 gas 会导致你的交易失败。
  // 状态更改被撤销。
  // 花费的 gas 不会被退还。
  function forever() public {
      // 在这里，我们运行一个循环直到花费所有的 gas
      // 交易失败
      while (true) {
          i += 1;
      }
  }
}
```
代码术语解释：

gas：以太坊中的计算单位。\
gas spent：在交易中使用的 gas 总数。\
gas price：为每单位 gas 所支付的以太币数量。\
gas limit：交易中可使用的最大 gas 数量，由用户设置。\
区块 gas 限制：每个区块中允许的最大 gas 数量，由网络设置。
