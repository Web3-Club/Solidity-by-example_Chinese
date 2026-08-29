# Solidity-by-example 中文翻译
[![开源授权](https://img.shields.io/github/license/Web3-Club/solidity-by-example_Chinese)](https://github.com/Web3-Club/solidity-by-example_Chinese)                                                                                      [![GitHub stars](https://img.shields.io/github/stars/Web3-Club/solidity-by-example_Chinese.svg?style=social&label=Stars)](https://github.com/Web3-Club/solidity-by-example_Chinese)                                   [![GitHub watchers](https://img.shields.io/github/watchers/Web3-Club/solidity-by-example_Chinese.svg?style=social&label=Watch)](https://github.com/Web3-Club/solidity-by-example_Chinese)<br>


<a href="https://twitter.com/intent/follow?screen_name=web3clubCN">
        <img src="https://img.shields.io/twitter/follow/web3clubCN?style=social&logo=X"
            alt="follow on Twitter"></a>

## 简介
[solidity-by-example](https://solidity-by-example.org/) 文档 简体中文翻译 项目<br>
当前对照英文原站 **v 0.8.26**（[solidity-by-example.org](https://solidity-by-example.org/)）校对并补齐。

Solidity是一种编程语言，专门用于在以太坊区块链上编写智能合约。智能合约是一种自动执行的合约，它们旨在简化和加速各种交易，例如资产转移、投票和奖励分配。Solidity可以让开发人员创建这些合约，并在以太坊网络上部署和运行它们。

Solidity是一种类似于JavaScript的面向对象语言，具有与其他高级编程语言相似的语法和结构。它支持许多计算机科学概念，例如继承、封装和抽象，同时还提供了特定于区块链的功能，例如以太币转账和智能合约事件。

使用Solidity，开发人员可以构建去中心化应用程序（DApps），这些应用程序可以为用户提供更安全、更透明和更可靠的服务。由于Solidity具有向后兼容性，因此开发人员可以轻松地升级他们的智能合约以实现更好的性能和更丰富的功能。

---
[solidity-by-example](https://solidity-by-example.org/) 是一个展示Solidity各种用法的仓库，本仓库是对于其英语版的翻译。 


## 🔖 目录

顺序与英文原站一致。每篇译文开头有「对应英文原页」链接。黑客攻击请以 **Unit5** 为准（Unit2 中仍保留若干早期误放入「应用」的旧稿）。

### 基础

1. [Hello World](./01%20Hello%20World.md)
2. [第一个应用](./02%20第一个应用.md)
3. [基本数据类型](./03%20基本数据类型.md)
4. [变量](./04%20变量.md)
5. [常量](./05%20常量.md)
6. [不可变](./06%20不可变.md)
7. [读写状态变量](./07%20读写状态变量.md)
8. [以太币和 Wei](./08%20以太币和Wei.md)
9. [Gas 与 Gas Price](./09%20Gas费.md)
10. [条件判断](./10%20条件判断.md)
11. [循环语句](./11%20循环语句.md)
12. [映射](./12%20映射.md)
13. [数组](./13%20数组.md)
14. [枚举](./14%20枚举.md)
15. [用户自定义值类型](./15%20用户自定义值类型.md)
16. [结构体](./16%20结构体.md)
17. [数据位置](./17%20数据位置.md)
18. [瞬时存储](./18%20瞬时存储.md)
19. [函数](./19%20函数.md)
20. [View 和 Pure 函数](./20%20View和Pure函数.md)
21. [错误或异常](./21%20错误或异常.md)
22. [函数修饰符](./22%20函数修饰符.md)
23. [事件](./23%20事件.md)
24. [高级事件](./24%20高级事件.md)
25. [构造函数](./25%20构造函数.md)
26. [继承](./26%20继承.md)
27. [继承状态变量的屏蔽](./27%20继承状态变量的屏蔽.md)
28. [调用父合约](./28%20调用父合约.md)
29. [可见性](./29%20可见性.md)
30. [接口](./30%20接口.md)
31. [Payable](./31%20Payable%20函数.md)
32. [发送以太币](./32%20发送以太币.md)
33. [回退函数](./33%20回退函数.md)
34. [Call](./34%20Call.md)
35. [Delegatecall](./35%20Delegatecall.md)
36. [函数选择器](./36%20函数选择器.md)
37. [调用其他合约](./37%20调用其他合约.md)
38. [从合约创建合约](./38%20从合约创建合约.md)
39. [Try / Catch](./39%20Try%20Catch.md)
40. [导入](./40%20导入.md)
41. [库](./41%20库.md)
42. [ABI 编码](./42%20ABI%20编码.md)
43. [ABI 解码](./43%20ABI%20解码.md)
44. [Keccak256 哈希](./44%20Keccak256%20哈希.md)
45. [验证签名](./45%20验证签名.md)
46. [Gas 优化](./46%20Gas%20优化.md)
47. [位运算](./47%20位运算.md)
48. [Unchecked Math](./48%20Unchecked%20Math.md)
49. [汇编变量](./49%20汇编变量.md)
50. [汇编条件语句](./50%20汇编条件语句.md)
51. [汇编循环](./51%20汇编循环.md)
52. [汇编错误](./52%20汇编错误.md)
53. [汇编数学运算](./53%20汇编数学运算.md)

### Unit2 应用

- [以太钱包](./Unit2%20应用/以太钱包.md)
- [多签钱包](./Unit2%20应用/多签钱包.md)
- [默克尔树](./Unit2%20应用/默克尔树.md)
- [可迭代映射](./Unit2%20应用/可迭代映射.md)
- [ERC20](./Unit2%20应用/ERC20.md)
- [ERC721](./Unit2%20应用/ERC721.md)
- [ERC1155](./Unit2%20应用/ERC1155.md)
- [无 Gas 代币转账](./Unit2%20应用/无Gas代币转账.md)
- [简单字节码合约](./Unit2%20应用/简单字节码合约.md)
- [使用 Create2 预计算合约地址](./Unit2%20应用/使用Create2预计算合约地址.md)
- [最小代理合约](./Unit2%20应用/最小代理合约.md)
- [可升级代理](./Unit2%20应用/可升级代理.md)
- [部署任意合约](./Unit2%20应用/部署任意合约.md)
- [写入任意存储槽](./Unit2%20应用/写入任意存储槽.md)
- [单向支付渠道](./Unit2%20应用/单向支付渠道.md)
- [双向支付渠道](./Unit2%20应用/双向支付渠道.md)
- [英式拍卖](./Unit2%20应用/英式拍卖.md)
- [荷兰式拍卖](./Unit2%20应用/荷兰式拍卖.md)
- [众筹基金](./Unit2%20应用/众筹基金.md)
- [多调用](./Unit2%20应用/多调用.md)
- [多委托调用](./Unit2%20应用/多委托调用.md)
- [定时锁](./Unit2%20应用/定时锁.md)
- [汇编中的二进制求幂](./Unit2%20应用/汇编中的二进制求幂.md)
- [默克尔空投](./Unit2%20应用/默克尔空投.md)

### Unit3 测试

- [使用 Echidna 测试智能合约](./Unit3%20测试/使用Echidna进行智能合约测试.md)

### Unit4 DeFi

- [01 Uniswap V2 Swap](./Unit4%20DeFi/01%20Uniswap%20V2%20Swap.md)
- [02 Uniswap V2 添加与移除流动性](./Unit4%20DeFi/02%20Uniswap%20V2%20Add%20Remove%20Liquidity.md)
- [03 Uniswap V2 最优单边供给](./Unit4%20DeFi/03%20Uniswap%20V2%20Optimal%20One%20Sided%20Supply.md)
- [04 Uniswap V2 Flash Swap](./Unit4%20DeFi/04%20Uniswap%20V2%20Flash%20Swap.md)
- [05 Uniswap V3 Swap](./Unit4%20DeFi/05%20Uniswap%20V3%20Swap.md)
- [06 Uniswap V3 流动性](./Unit4%20DeFi/06%20Uniswap%20V3%20Liquidity.md)
- [07 Uniswap V3 闪电贷](./Unit4%20DeFi/07%20Uniswap%20V3%20Flash%20Loan.md)
- [08 Uniswap V3 Flash Swap 套利](./Unit4%20DeFi/08%20Uniswap%20V3%20Flash%20Swap%20Arbitrage.md)
- [09 Uniswap V4 Swap](./Unit4%20DeFi/09%20Uniswap%20V4%20Swap.md)
- [10 Uniswap V4 闪电贷](./Unit4%20DeFi/10%20Uniswap%20V4%20Flash%20Loan.md)
- [11 Uniswap V4 限价单](./Unit4%20DeFi/11%20Uniswap%20V4%20Limit%20Order.md)
- [12 Chainlink 价格预言机](./Unit4%20DeFi/12%20Chainlink%20Price%20Oracle.md)
- [13 Chronicle 价格预言机](./Unit4%20DeFi/13%20Chronicle%20Price%20Oracle.md)
- [14 DAI Proxy](./Unit4%20DeFi/14%20DAI%20Proxy.md)
- [15 质押奖励](./Unit4%20DeFi/15%20Staking%20Rewards.md)
- [16 离散质押奖励](./Unit4%20DeFi/16%20Discrete%20Staking%20Rewards.md)
- [17 Vault](./Unit4%20DeFi/17%20Vault.md)
- [18 Token Lock](./Unit4%20DeFi/18%20Token%20Lock.md)
- [19 恒定和 AMM](./Unit4%20DeFi/19%20Constant%20Sum%20AMM.md)
- [20 恒定乘积 AMM](./Unit4%20DeFi/20%20Constant%20Product%20AMM.md)
- [21 Stable Swap AMM](./Unit4%20DeFi/21%20Stable%20Swap%20AMM.md)

### Unit5 黑客攻击及预防

- [重入攻击](./Unit5%20黑客攻击及预防/重入攻击.md)
- [算术溢出与下溢](./Unit5%20黑客攻击及预防/算术溢出.md)
- [自毁函数](./Unit5%20黑客攻击及预防/自毁函数.md)
- [访问私有数据](./Unit5%20黑客攻击及预防/访问私有数据.md)
- [Delegatecall 攻击](./Unit5%20黑客攻击及预防/delegatecall.md)
- [随机数预测](./Unit5%20黑客攻击及预防/随机数预测.md)
- [拒绝服务攻击](./Unit5%20黑客攻击及预防/拒绝服务攻击.md)
- [tx.origin 钓鱼](./Unit5%20黑客攻击及预防/tx.origin钓鱼.md)
- [使用外部合约隐藏恶意代码](./Unit5%20黑客攻击及预防/使用外部合约隐藏恶意代码.md)
- [蜜罐](./Unit5%20黑客攻击及预防/蜜罐.md)
- [抢跑](./Unit5%20黑客攻击及预防/抢跑.md)
- [区块时间戳操纵](./Unit5%20黑客攻击及预防/区块时间戳操纵.md)
- [签名重放](./Unit5%20黑客攻击及预防/签名重放.md)
- [绕过合约大小检查](./Unit5%20黑客攻击及预防/绕过extcodesize.md)
- [在同一地址部署不同合约](./Unit5%20黑客攻击及预防/在同一个地址部署不同的合约.md)
- [金库通货膨胀攻击](./Unit5%20黑客攻击及预防/金库通货膨胀攻击.md)
- [WETH Permit](./Unit5%20黑客攻击及预防/WETH%20Permit.md)
- [63 / 64 Gas 规则](./Unit5%20黑客攻击及预防/63-64%20Gas规则.md)

### Unit6 EVM

- [EVM 存储布局](./Unit6%20EVM/存储布局.md)
- [EVM 内存布局](./Unit6%20EVM/内存布局.md)

### Unit7 Foundry

- [01 基础](./Unit7%20Foundry/01%20基础.md)
- [02 授权](./Unit7%20Foundry/02%20授权.md)
- [03 错误](./Unit7%20Foundry/03%20错误.md)
- [04 事件](./Unit7%20Foundry/04%20事件.md)
- [05 发送](./Unit7%20Foundry/05%20发送.md)
- [06 时间](./Unit7%20Foundry/06%20时间.md)
- [07 签名](./Unit7%20Foundry/07%20签名.md)
- [08 标签](./Unit7%20Foundry/08%20标签.md)
- [09 Mock Call](./Unit7%20Foundry/09%20Mock%20Call.md)
- [10 Store](./Unit7%20Foundry/10%20Store.md)

早期施工计划见 **[issues#2](https://github.com/Web3-Club/solidity-by-example_Chinese/issues/2)**。

## 更新日志

        2023/03/19-2023/06/26 完成  Hello World - Unchecked Math 板块翻译
                              完成  Application 部分的  Ether Wallet -  Precompute Contract Address with Create2 板块翻译
        2023/10/11-2023/10/12 对部分板块进行解释扩充和代码实例的加入
        2024/02/21 优化目录
        2024/06/21 完成 folder：应用，Unit5 黑客攻击及预防
                   name：双向支付渠道 众筹基金、英式拍卖、荷兰式拍卖、 多调用，多委托调用，定时锁，汇编中的二进制求幂（应用），重入攻击（黑客攻击及预防）
        2026/08/25 对照 solidity-by-example.org v0.8.26 校对已有译文，
                   补齐条件判断、瞬时存储、高级事件、Call/Delegatecall、汇编、
                   ERC721/ERC1155、默克尔空投、63/64 Gas 规则、EVM、Foundry、
                   Uniswap V3/V4 及其余 DeFi 等全部缺页；示例合约版本升至 0.8.26

<br>

## ❤️ 项目贡献者
**永远感谢他们为本社区下项目所作出的贡献!**

[![contrib graph](https://contrib.rocks/image?repo=Web3-Club/solidity-by-example_Chinese)](https://github.com/Web3-Club/solidity-by-example_Chinese/graphs/contributors)

<br>

## 💐 赞助我们 
### 通过Donate3


<a href="https://www.donate3.xyz/donateTo?cid=bafkreif5ecvwp7vanir2geib43nws7zvaac46rvlryzwwm47knutcv6xee" target="_blank"><img src="https://www.donate3.xyz/Donate3ToMe.svg" alt="Donate3 To Me"></a>

### Ethereum

                        0x663d5dafe4362927e6dab344e8953b0ad4439d3f


### **社群宗旨**   
#### **永远关注知识和技术的进步，而不是价格**<br>   
在此，我们希望为所有的对Web3未来感兴趣和欲为其“添砖加瓦”的朋友们一起,创造出更美好的Web3未来前景！<br>  
（详见[关于我们](https://github.com/Web3-Club/Intro.#%E7%AE%80%E4%BB%8B) ）


## Star History
[![Star History Chart](https://api.star-history.com/svg?repos=Web3-Club/Solidity-by-example_Chinese&type=Date)](https://star-history.com/#Web3-Club/Solidity-by-example_Chinese/&Date)


# ⚠️ 免责声明

The organization that developed this project, "Web3Club", is currently a non-profit open source community, not a company or corporationand.

All translations of the project were developed by members and contributors to the project, and any content in the project is protected by an open source licence，

We always open source the original open source project in accordance with the license of the original open source project before translation.And in accordance with the requirements of the licence,the information of the original English project or the original author will be indicated in the following sections.

If you have any questions about licence or copyright, please read the LICENCE section below or contact us at web3clubCN@outlook.com

<br>

## 📖 LICENCE
### [Attribution-ShareAlike 4.0 International](https://creativecommons.org/licenses/by-sa/4.0/legalcode)<br>
Built by China Web3-Club [contributors](https://github.com/Web3-Club/solidity-by-example_Chinese/graphs/contributors) with heart. <br> 
Copyright © [solidity-by-example.org](https://solidity-by-example.org/)｜[@solidity-by-example](https://github.com/solidity-by-example)<br> 
Chinese Translation copyright © 2023-2026 &emsp; [Web3-Club](https://github.com/Web3-Club)<br> 
ALL RIGHT RESERVED  



