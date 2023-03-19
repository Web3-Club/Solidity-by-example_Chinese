# 以太币和Wei

交易使用以太币支付。

类似于一个美元等于100分，一个以太币等于10^18 Wei。
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

contract EtherUnits {
uint public oneWei = 1 wei;
// 1 Wei 等于 1
bool public isOneWei = 1 wei == 1;

csharp
Copy code
uint public oneEther = 1 ether;
// 1 以太币等于10^18 Wei
bool public isOneEther = 1 ether == 1e18;
}
```
## 术语解释

### 以太币（Ether）
以太币是以太坊网络的本地加密货币，用于支付交易费用和奖励矿工。\
### Wei
Wei是以太币的最小单位，是以太币的十八个数量级（10的18次方）。\
### 交易费用（Transaction fees）
交易费用是指在以太坊网络中发送交易时需要支付的以太币数量，也称为燃气费用。\
### 矿工奖励（Miner rewards）
矿工奖励是指在以太坊网络中打包交易和验证区块链的矿工可以获得的以太币奖励。
