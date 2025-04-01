# BSC 上的 MEV 攻击

<h4 align="center">
    <p>
        <a href="https://github.com/DonggeunYu/MEV-Attack-on-the-BSC/blob/main/README.md">English</a> |
        <a href="https://github.com/DonggeunYu/MEV-Attack-on-the-BSC/blob/main/README_ko.md">한국어</a> |
        <b>中文</b>
    </p>
</h4>

## 概述
本仓库实现了如何在币安智能链(BSC)网络上对去中心化交易所(DEX)进行最大可提取价值(MEV)攻击来获取利润。
- 由于 MEV 开发直接与金钱收益相关,很难找到真正有帮助的资源。我希望这个资源能启发其他开发者并为他们的项目增添实际价值。
- 该项目在2024年1月至2024年5月期间开发并运行。
- 技术问题和反馈请使用仓库的 Issues 页面,其他咨询请联系 ddonggeunn@gmail.com。

<br>
<br>
<br>

## 目录
- [BSC 上的 MEV 攻击](#bsc-上的-mev-攻击)
  * [概述](#概述)
  * [目录](#目录)
  * [三明治攻击交易示例](#三明治攻击交易示例)
    + [使用 bloXroute 的三明治攻击](#使用-bloxroute-的三明治攻击)
    + [使用 48 Club 的三明治攻击](#使用-48-club-的三明治攻击)
    + [使用普通方式的三明治攻击](#使用普通方式的三明治攻击)
  * [MEV 理论解释](#mev-理论解释)
    + [缩写](#缩写)
    + [预期的区块链理想流程](#预期的区块链理想流程)
    + [DEX 交换的漏洞](#dex-交换的漏洞)
    + [如何防范 MEV 攻击者](#如何防范-mev-攻击者)
      - [区块提议者和构建者是否分离?](#区块提议者和构建者是否分离)
  * [源代码和公式的详细解释](#源代码和公式的详细解释)
    + [项目结构](#项目结构)
      - [目录结构](#目录结构)
    + [IAC 提示](#iac-提示)
    + [关于 MEV-Relay](#关于-mev-relay)
    + [解释 Solidity 代码](#解释-solidity-代码)
      - [sandwichFrontRun 和 sandwichFrontRunDifficult 有什么区别?](#sandwichfrontrun-和-sandwichfrontrundifficult-有什么区别)
      - [关于 BackRun 函数](#关于-backrun-函数)
      - [交换路由器](#交换路由器)
      - [支持的 DEX 列表](#支持的-dex-列表)
      - [Solidity 测试代码](#solidity-测试代码)
    + [解释 Python 代码](#解释-python-代码)
      - [重要说明](#重要说明)
      - [书面过程说明](#书面过程说明)
      - [为什么套利只用于普通路径?](#为什么套利只用于普通路径)
      - [追踪交易](#追踪交易)
    + [公式部分](#公式部分)
      - [理解 DEX AMM 的重要性](#理解-dex-amm-的重要性)
      - [Uniswap V2、V3 的结构](#uniswap-v2v3-的结构)
      - [将 Uniswap V2 交换优化到极致](#将-uniswap-v2-交换优化到极致)
        * [1. Uniswap V2 输出金额公式](#1-uniswap-v2-输出金额公式)
        * [2. 分析 Uniswap V2 交换](#2-分析-uniswap-v2-交换)
        * [3. 交换公式优化](#3-交换公式优化)
        * [4. 极致优化公式的验证](#4-极致优化公式的验证)
      - [多跳套利公式](#多跳套利公式)
        * [1. 2跳套利公式](#1-2跳套利公式)
        * [2. 多跳套利公式](#2-多跳套利公式)
  * [EigenPhi 被过度炒作](#eigenphi-被过度炒作)
    + [当三明治攻击被识别为套利时](#当三明治攻击被识别为套利时)
    + [使用 48 Club 进行三明治攻击时](#使用-48-club-进行三明治攻击时)
  * [常见问题](#常见问题)
      - [为什么选择 BSC 网络?](#为什么选择-bsc-网络)
      - [你未来是否计划继续开发套利或 MEV?](#你未来是否计划继续开发套利或-mev)
  * [更多信息](#更多信息)

<small><i><a href='http://ecotrust-canada.github.io/markdown-toc/'>目录由 markdown-toc 生成</a></i></small>

## MEV 理论解释

### 缩写
- EL: 执行层(Execution Layer)
- CL: 共识层(Consensus Layer)
- DEX: 去中心化交易所(Decentralized Exchange)
- MEV: 最大可提取价值(Maximal Extractable Value)

<br>
<br>

### 预期的区块链理想流程
本节主要是理论性的,只包含开发 MEV 攻击所需的核心信息。具体实现细节请参考其他章节。

从理论上讲,区块链上预期的区块创建过程如下:

1. 用户交易创建:
  - 用户创建交易并使用其钱包的私钥签名。
2. 交易提交:
  - 已签名的交易通过远程过程调用(RPC)或其他通信协议发送到执行层(EL)节点(如 Geth、Erigon)。
3. 交易验证和汇集:
  - EL 节点验证交易并将其添加到本地交易池(也称为内存池)中。
4. 点对点同步:
  - EL 节点在去中心化网络中与对等节点同步,共享内存池。注意,由于网络延迟和同步延迟,内存池的内容可能在所有节点上并不完全一致。
5. 验证者选择:
  - 信标链随机选择一个验证者(A)进行区块生产。这个选择过程涉及随机性,每个验证者被选中的机会与其质押的 ETH 数量成正比。
6. 区块构建:
  - 验证者(A)从 EL 节点获取交易并构建区块。区块的构建是为了通过包含手续费更高的交易来最大化利润。EL 和共识层(CL)集成以交换必要的信息。如果用户提交给 EL 的交易没有到达验证者(A)的 EL 节点,它可能不会被包含在区块中。
7. 收集证明:
  - 验证者(A)准备区块并寻求其他验证者(B)的证明。这些证明确认其他验证者同意验证者(A)提议的区块。
8. 区块验证和证明提交:
  - 验证者(B)审查提议的区块并验证其正确性。然后他们提交证明以支持该区块。
9. 区块最终确定:
  - 一旦收集到足够的证明,区块就会被添加到信标链中。这一步通常涉及基于共识规则和所需证明数量来最终确定区块。

<br>
<br>

### DEX 交换的漏洞
在区块链上,DEX 使得交换各种代币变得更容易。然而,使用 DEX 交换代币的用户面临着资产损失的风险。这个漏洞的关键在于区块内交易的顺序,这可能被操纵从而产生安全漏洞。

**套利攻击**

- 定义: 套利是一种利用两个交易所之间价格差异来获取利润的技术。
- 过程
  1. 当有人在去中心化交易所(DEX)进行交易时,会发生价格变化。
  2. 你可以检测到价格变化并在下一个区块中提交交易,利用价格差异来获取利润。
- 然而,要获得利润,你必须始终领先于其他人。
- 优化过程
  1. 你可以分析内存池中其他人提交的交易,内存池是尚未成为区块一部分的交易集合。你可以计算交易是否调用 DEX 以及预期的价格变化会是什么。
  2. 一旦分析了交易,你就可以计算利用价格变化机会的最大预期利润。
  3. 攻击者的交易必须紧跟在受害者的交易之后,以击败其他攻击者的竞争并获取利润。

**三明治攻击**

![https://medium.com/coinmonks/demystify-the-dark-forest-on-ethereum-sandwich-attacks-5a3aec9fa33e](images/sandwich.jpg)

[图片来源](https://medium.com/coinmonks/demystify-the-dark-forest-on-ethereum-sandwich-attacks-5a3aec9fa33e)

- 定义: 一种通过在受害者交易前后放置攻击交易来操纵 DEX 池流动性以获取利润的技术。
- 注意: 这种技术比套利更复杂,与套利不同,它会导致受害者的货币损失。三明治攻击的受害者实际上会遭受资产损失。
- 过程
  1. 攻击者通过在受害者交易之前放置前置交易来人为抬高价格。(在这个过程中,价格上涨越大,攻击者可能也会损失越多的钱。)
  2. 受害者以高于交易前预期的价格进行交易。此时,价格上涨等于前置交易量与受害者交易量之间的差额。
  3. 攻击者在受害者交易之后直接放置后置交易。这使攻击者能够收回前置过程中产生的损失并获得额外利润。

[未完待续...] 

### 如何防范 MEV 攻击者
MEV 攻击对区块链生态系统有严重的负面影响,但由于区块链的结构限制,很难从根本上解决这个问题。目前只有一些部分缓解这些问题的技术。

MEV 攻击者需要知道受害者的交易信息才能发起攻击,而这些信息在内存池阶段就会暴露给攻击者。因此,可以使用以下策略来保护受害者的交易。

步骤

1. 选择区块提议者和区块构建者来构建提议的区块。
2. 受害者将交易提交到私有内存池而不是公共内存池。
3. 攻击者分析公共或私有内存池中的受害者交易以识别获利机会。
4. 攻击者将其前置和后置交易与受害者的交易打包在一起,并将它们提交到私有订单流。
5. 该过程涉及攻击者返还从受害者获得的部分利润(例如,攻击者可能以 gas 费用的形式提交 MEV 费用或向特定合约地址支付费用)。
6. 验证打包的执行情况和要支付的费用的适当性。如果对同一受害者交易提交了多个打包,则选择费用最高的打包。
7. 区块提议者查找公共内存池和私有订单流中的交易。然后组织并提议一个能为验证者带来最高收益的区块。
8. 被选中的验证者构建区块提议者提议的区块。
9. 受害者收到攻击者在步骤5中返还的部分利润。

这些服务使所有参与者受益。

- 受害者: 可以减少损失。
- 攻击者: 通过将交易顺序固定为一个打包,他们可以消除由于交易顺序而导致的失败和错误风险。错误风险被消除是因为当你提交打包时,你会测试它以确保它运行时没有错误。(但是,你可能会因为必须向受害者返还一些利润而产生一些损失)。
- 验证者: 你可以最大化你的利润。(Flashbots 团队[声称](https://hackmd.io/@flashbots/mev-in-eth2)使用 MEV-Boost 的验证者可以将其利润提高60%以上)。

[Flashbots on ETH](https://www.flashbots.net)、[bloXroute on BSC](https://bloxroute.com) 和 [48 Club on BSC](https://docs.48.club/buidl/infrastructure/puissant-builder) 等服务提供这种功能。然而,对于 bloXroute,有人声称它破坏了 BSC 生态系统(详见下面的[更多信息](#更多信息)页面)。

<br>

#### 区块提议者和构建者是否分离?
是的。将区块提议者和构建者分离的概念被称为提议者/构建者分离(PBS)。

从以太坊的 MEV-Boost 开始是一个很好的起点。

![https://docs.flashbots.net/flashbots-mev-boost/introduction](images/mev-boost-integration-overview.jpg)

PBS 通过区块构建者之间的竞争促进了以太坊网络的去中心化,增加了网络参与者的整体收益。

<br>
<br>
<br> 

## 源代码和公式的详细解释
由于使用节点服务器和 bloXroute 的成本很高,加上需要快速看到结果,我们没有给项目结构足够的思考或重构时间。如果代码难以阅读,我深表歉意。🙇

### 项目结构
#### 目录结构
- **contract**: 这是一个基于 **Hardhat** 的 Solidity 项目。
  * **contracts**: 包含执行 **MEV** 的 Solidity 代码。
  * **test**: 我用 TypeScript 编写了 Solidity 测试代码。

- **docker**
  * **node**: 运行节点所需的 Dockerfile。
  * **source**: 用于运行 Python 源代码的 Dockerfile(生产环境未使用)。

- **iac**: 用 **Pulumi** 编写的用于运行节点的基础设施即代码(IAC)。
  * **aws**: AWS 环境中的 IAC。
    + **gcp**: GCP 环境中的 IAC(开发中已弃用)。

- **pyrevm**: 这是 **以太坊虚拟机**(EVM)的 Rust 实现 REVM 的 Python 包装器。原始代码可在 [paradigmxyz/pyrevm](https://github.com/paradigmxyz/pyrevm) 找到。

- **src**: 用于捕获 MEV 机会的 Python 代码。
  * **dex**: 这是 DEX AMM 的 Python 实现的初始代码。现在已作为遗留代码弃用,我们已完全实现了 Uniswap V2 和 V3 的 AMM。对于 Uniswap V3,我们参考了 [ucsb-seclab/goldphish](https://github.com/ucsb-seclab/goldphish/blob/main/pricers/uniswap_v3.py)。Curve 的 AMM 在每个池的计算中有 ±1 的误差。有关支持的 Curve 池的详细信息,请参见 `tests/test_dex/test_curve_*.py`。
  * **multicall**: 提供 MultiCall 功能。原始代码可在 [banteg/multicall.py](https://github.com/banteg/multicall.py) 找到。

- **tests**: 这是用于测试 `src` 代码的代码。

<br>
<br>

### IAC 提示

- 对于区域,我推荐美国纽约或德国。
  * 通过将你的节点尽可能靠近其他公开接收交易的节点,你可以在有限的时间内快速获得共享的内存池。

- 不要将执行 MEV 攻击的客户端和节点与负载均衡器等分离。
  * 虽然你可能想要为 RPC 服务器的安全性去中心化你的网络,但 MEV 客户端和节点之间的网络速度至关重要。
  * 我在同一个 EC2 实例上运行 MEV 客户端和节点,以最小化网络损失。

- 积极使用快照功能很重要。
  * `geth` 的可靠性可能稍差。
  * 我有多次在 geth 崩溃时链数据损坏的经历,所以我建议积极使用快照功能。

<br>
<br>

### 关于 MEV-Relay

该项目支持三种路径

- **普通**: 不使用 MEV-Relay,通过公共内存池。
- **48 Club**: 使用 Puissant API 执行 MEV 操作。
- **bloXroute**: 有基础成本,且每个打包都需要支付单独的费用。

|名称|支持打包|基础价格|
|------|:---:|---|
|普通|X|-|
|[48 Club](https://docs.48.club/buidl/infrastructure/puissant-builder/send-bundle)|O|-|
|[bloXroute](https://docs.bloxroute.com/apis/mev-solution/bsc-bundle-submission)|O|$5,000/月|
|[TxBoost](http://txboost.com)|O|-|

<br>

在[三明治攻击交易示例](#三明治攻击交易示例)中,你可以看到与普通路径相比,bloXroute 和 48 Club 的利润较低。由于 MEV 攻击不是孤立进行的,不可避免地会与其他竞争者竞争,这会带来成本。

首先,让我们分析普通路径: 普通路径不使用 MEV-Relay,所以在攻击过程中不能保证交易执行和顺序。如果你未能按正确的交易顺序执行 DEX 交易,你可能会成为其他攻击者的受害者。如果你在 MEV 攻击完成后尝试攻击,你可能会损失交易费用。此外,如果你试图攻击的池中的代币数量发生变化,交易可能会失败(假设另一个攻击者已经尝试过)。即使失败,你仍然会产生 gas 成本。

接下来,让我们看看 bloXroute 和 48 Club。使用 MEV-Relay 有两个主要优势。首先,你可以保证交易的顺序,其次,如果交易失败,你不会被收费。要消除这种风险,你需要支付去风险费用。去风险的成本基本上包括你支付给 MEV-Relay 的费用和赢得竞争的投标成本。

因此,在分析单个交易时,普通路径的成本可能并不明显。

更多信息

- [BSC MEV Overview on the bnbchain docs](https://docs.bnbchain.org/bnb-smart-chain/validator/mev/overview/)
- [bloxroute exploiting users for MEV on the bnb-chain/bsc GitHub issue](https://github.com/bnb-chain/bsc/issues/1706)

[未完待续...] 

### 解释 Solidity 代码

#### sandwichFrontRun 和 sandwichFrontRunDifficult 有什么区别?
假设存在池 A 和池 B。

- **sandwichFrontRun**: 交易按以下顺序执行 `我的合约 -> A 池 -> 我的合约 -> B 池 -> 我的合约`。

- **sandwichFrontRunDifficult**: 交易按以下顺序执行: `我的合约 -> A 池 -> B 池 -> 我的合约`。在这种情况下,每个池都需要验证它可以向下一个池发送代币,并且下一个池可以接收代币,这可以减少代币转移的次数。

<br>

#### 关于 BackRun 函数
有三个函数

- **sandwichBackRunWithBloxroute**: 此函数执行后置交易,然后通过 bloXroute 支付费用。

- **sandwichBackRun**: 此函数执行后置交易。它在功能上等同于 `sandwichBackRunWithBloxroute`,只是不涉及支付费用。

- **sandwichBackRunDifficult**: 此函数执行类似于 `sandwichBackRun` 的后置交易,但要求每个池在交易进行之前验证它可以发送和接收代币。

#### 交换路由器
合约 `contract/contracts/dexes/SwapRouter.sol` 在高层次上促进交换。该合约只需要 DEX ID、池地址、代币地址和金额信息就可以执行交换。

<br>

#### 支持的 DEX 列表
交换支持以下 DEX

- **Uniswap V2 系列**: UniswapV2、SushiswapV2、PancakeSwapV2、THENA、BiswapV2、ApeSwap、MDEX、BabySwap、Nomiswap、WaultSwap、GibXSwap
- **Uniswap V3 系列**: UniswapV3、SushiswapV3、PancakeSwapV3、THENA FUSION
- **Curve (ETH)**: 支持大多数主要池。

<br>

#### Solidity 测试代码

编写 Solidity 测试代码时,我考虑了以下几点:

- 确保 DEX 交换正常工作
- 连续跨多个池交换时的连接状态(套利)
- 适用于不同链
- 利用 [hardhat-exposed](https://www.npmjs.com/package/hardhat-exposed) 测试私有函数和公共函数

测试代码位于以下文件:

- `contract/test/dexes.ts`
- `contract/test/ArbitrageSwap.ts`

<br>
<br>

## 公式部分

#### 理解 DEX AMM 的重要性
理解 DEX 的自动做市商(AMM)对于开发 MEV 攻击至关重要。这是因为 AMM 的设计直接影响了价格变化的计算方式。

#### Uniswap V2、V3 的结构
Uniswap V2 和 V3 是目前最流行的 DEX 协议。它们的主要区别在于:

- V2 使用恒定乘积公式 x*y=k
- V3 引入了集中流动性的概念

#### 将 Uniswap V2 交换优化到极致

##### 1. Uniswap V2 输出金额公式
基本公式:
- $x * y = k$ (恒定乘积公式)
- $x$: 代币 X 的储备量
- $y$: 代币 Y 的储备量
- $k$: 常数

当用户想要用 $\Delta x$ 数量的代币 X 换取代币 Y 时:
- $(x + \Delta x)(y - \Delta y) = k$
- $\Delta y = \frac{y\Delta x}{x + \Delta x}$

##### 2. 分析 Uniswap V2 交换
考虑手续费:
- 实际输入量 = 输入量 * (1 - 费率)
- 费率通常为 0.3% (0.003)
- $\Delta y = \frac{0.997y\Delta x}{x + 0.997\Delta x}$

##### 3. 交换公式优化
对于三明治攻击,我们需要:
1. 计算最优前置交易金额
2. 估算受害者交易的影响
3. 计算后置交易的收益

设:
- $n$: 池的基数(通常为1000)
- $s$: 滑点(通常为3)
- $R_{1,in}$: 第一个池的输入储备
- $R_{1,out}$: 第一个池的输出储备
- $R_{2,in}$: 第二个池的输入储备
- $R_{2,out}$: 第二个池的输出储备

##### 4. 极致优化公式的验证
最优前置交易金额($x^*$)的计算:

- $k = (n - s)n^2R_{2,in} + (n - s)^2nR_{1,out}$
- $a = k^2$
- $b = 2n^3R_{1,in}R_{2,in}k$
- $c = (n^3R_{1,in}R_{2,in})^2 - (n - s)^3n^3R_{1,in}R_{2,in}R_{1,out}R_{2,out}$
- $x^* = \frac{-b + \sqrt{b^2 - 4ac}}{2a}$

#### 多跳套利公式

##### 1. 2跳套利公式
对于两个池之间的套利:

- $(n^2R_{1,in}R_{2,in})^2 + 2n^2R_{1,in}R_{2,in}kx + (kx)^2 = (n - s)^2n^2R_{1,in}R_{2,in}R_{1,out}R_{2,out}$
- $k^2x^2 + 2n^2R_{1,in}R_{2,in}kx + (n^2R_{1,in}R_{2,in})^2 - (n - s)^2n^2R_{1,in}R_{2,in}R_{1,out}R_{2,out} = 0$
- $a = k^2$
- $b = 2n^2R_{1,in}R_{2,in}k$
- $c = (n^2R_{1,in}R_{2,in})^2 - (n - s)^2n^2R_{1,in}R_{2,in}R_{1,out}R_{2,out}$
- $x^* = \frac{-b + \sqrt{b^2 - 4ac}}{2a}$

公式验证:
- 考虑两个池之间有2倍价格差的情况
- 变量:
  * $n = 1000$
  * $s = 3$
  * $R_{1,in} = 100 * 10^{18}$
  * $R_{1,out} = 1000 * 10^{18}$
  * $R_{2,in} = 1000 * 10^{18}$
  * $R_{2,out} = 200 * 10^{18}$
- [在 desmos 上计算和绘图](https://www.desmos.com/calculator/ltanp7fyvt)
  * 从下图可以看出,套利的预期收益为 $8.44176 \times 10^{18}$。我们还可以看到,为获得最大利润而应该投入第一个池的代币数量为 $20.5911 \times 10^{18}$。这些值与根公式得出的结果一致,因此我们可以说该公式已得到验证。

![](images/2-hop-arbitrage-formula.png)

##### 2. 多跳套利公式
如果有 $n$ 个池而不是两个池呢? 将上面的套利公式推广到 $n$ 个池的公式。

3跳:
- $k = (n - s)n^2R_{2,in}R_{3,in} + (n - s)^2nR_{1,out}R_{3,in} + (n-s)^3R_{1,out}R_{2,out}$
- $a = k^2$
- $b = 2n^3R_{0,in}R_{1,in}R_{2,in}k$
- $c = (n^3R_{1,in}R_{2,in}R_{3,in})^2 - (n - s)^3n^3R_{1,in}R_{2,in}R_{3,in}R_{1,out}R_{2,out}R_{3,out}$
- $x^* = \frac{-b + \sqrt{b^2 - 4ac}}{2a}$

4跳:
- $k = (n - s)n^3R_{2,in}R_{3,in}R_{4,in} + (n - s)^2n^2R_{1,out}R_{3,in}R_{4,in} + (n-s)^3nR_{1,out}R_{2,out}R_{4,in} + (n - s)^4R_{1,out}R_{2,out}R_{3,out}$
- $a = k^2$
- $b = 2n^4R_{1,in}R_{2,in}R_{3,in}R_{4,in}k$
- $c = (n^4R_{1,in}R_{2,in}R_{3,in}R_{4,in})^2 - (n - s)^4n^4R_{1,in}R_{2,in}R_{23,in}R_{4,in}R_{1,out}R_{2,out}R_{3,out}R_{4,out}$
- $x^* = \frac{-b + \sqrt{b^2 - 4ac}}{2a}$

公式推广:
- $h$ = 跳数
- $k = (n - s)n^h\prod_{i=2}^{h}R_{i,in} + \sum_{j=2}^{h}[(n - s)^jn^{h-j}\prod_{i=1}^{j-1}R_{i,out}\prod_{i=1}^{h-j}R_{i+j,in}]$
- $a = k^2$
- $b = 2n^h\prod_{i=1}^{h}R_{i,in}k$
- $c = (n^h\prod_{i=1}^{h}R_{i,in})^2 - (n - s)^hn^h\prod_{i=1}^{h}R_{i,in}\prod_{i=1}^{h}R_{i,out}$
- $x^* = \frac{-b + \sqrt{b^2 - 4ac}}{2a}$

公式实现:
```python
def get_multi_hop_optimal_amount_in(data: List[Tuple[int, int, int, int]]):
    """
    data: List[Tuple[int, int, int, int]]
        Tuple of (N, S, reserve_in, reserve_out)
    """
    h = len(data)
    n = 0
    s = 0
    prod_reserve_in_from_second = 1
    prod_reserve_in_all = 1
    prod_reserve_out_all = 1
    for idx, (N, S, reserve_in, reserve_out) in enumerate(data):
        if S > s:
          n = N
          s = S
        
        if idx > 0:
          prod_reserve_in_from_second *= reserve_in
        
        prod_reserve_in_all *= reserve_in
        prod_reserve_out_all *= reserve_out
    
    sum_k_value = 0
    for j in range(1, h):
      prod_reserve_out_without_latest = prod([r[3] for r in data[:-1]])
      prod_reserve_in_ = 1
      for i in range(0, h-j - 1):
        prod_reserve_in_ *= data[i + j + 1][2]
      sum_k_value += (n - s) ** (j + 1) * n ** (h - j - 1) * prod_reserve_out_without_latest * prod_reserve_in_
    k = (n - s) * n ** (h - 1) * prod_reserve_in_from_second + sum_k_value
    
    a = k ** 2
    b = 2 * n ** h * prod_reserve_in_all * k
    c = (n ** h * prod_reserve_in_all ) ** 2 - (n - s) ** h * n ** h * prod_reserve_in_all * prod_reserve_out_all
    
    numerator = -b + math.sqrt(b ** 2 - 4 * a * c)
    denominator = 2 * a
    return math.floor(numerator / denominator)
```

## EigenPhi 被过度炒作

### 当三明治攻击被识别为套利时
[EigenPhi](https://eigenphi.io) 是一个允许你追踪和分析 MEV 攻击的网络服务。主页显示了每个地址的利润。例如,它说一个地址在一周内获得了 $30,803,554.37 的利润。这是真的吗?

![](images/eigenphi_1.png)

<br>

如果你点击那个地址,你会看到 MEV 攻击的历史记录。让我们分析其中一次攻击。

![](images/eigenphi_2.png)

<br>

分析下面的代币流向。
![](images/eigenphi_3.png)
[0xadd318f803ff19bd5fc60a719bd9857610100066cb0e96108f2c1b0cbf74f7a5](https://eigenphi.io/mev/bsc/tx/0xadd318f803ff19bd5fc60a719bd9857610100066cb0e96108f2c1b0cbf74f7a5)

代币流向:

- 814.37543 WBNB -> 0x6D...8c81B(Cake-LP) -> 19,766,994,987.85470 CAT -> 0x58...eA735(PancakeV3Pool) -> 1424.92365 WBNB
- 5.26888 WBNB -> 0x6D...8c81B(Cake-LP) -> 118479117.67801 CAT -> 0x58...eA735(PancakeV3Pool) -> 5.28327 WBNB
- 0.00823 WBNB -> 0x6D...8c81B(Cake-LP) -> 185044.49375 CAT -> 0x58...eA735(PancakeV3Pool) -> 0.00823 WBNB
- 2.96989 BNB -> 0x48...84848

<br>

在上述交易中,区块中的位置是第3位。第2位很可能是受害者的交易。让我们找到第1位的交易。你可以在[这个链接](https://eigenphi.io/mev/eigentx/0xe5611d60eb6105d1d45aeeb90b09f8e309fc185cf679998e6ef1de97271b1eca)找到第1位的交易。

<br>

![](images/eigenphi_4.png)

你看到了我们在第3位交易中看到的相同数字吗? 那是在调整流动性深度。被标记为套利的攻击实际上是一次三明治攻击。让我们实际计算一下利润。

$1424.92365 \text{ WBNB} - 1421.26829 \text{ WBNB} + (5.28327 \text{ WBNB} - 5.26888 \text{ WBNB}) - 2.96989 \text{ BNB} = 0.69986 \text{ WBNB}$

攻击者的实际利润是 $419.916 ($0.69986 \times 600$),而不是 EigenPhi 显示的 $821,381.16。

<br>
<br>

### 使用 48 Club 进行三明治攻击时

记录在[这笔交易](https://eigenphi.io/mev/bsc/tx/0x8231ec4d8105694d404ec64ce7f08807c86f7a8dcb4e90dbbeee50ee8ae98110)中的攻击是一次三明治攻击。这笔交易在扣除成本后的收益为 $22.455143。

![](images/eigenphi_5.png)

查看区块中的所有交易,在位置0有一笔可疑交易,如下所示。这笔交易之所以可疑,是因为发送地址和接收地址相同,并且发送地址与执行三明治攻击的交易的发送地址相匹配。

<br>

![](images/eigenphi_6.png)

[区块中的交易](https://eigenphi.io/mev/eigentx/0x6ab43c8eda05d9ff09f11fd466d991bf4c98b7ae90208e0dc6c92a8470aa45d1,0x652dd0c03f1111611506fda141f9453fcc9d09919198c8ce2550086ae2bd92e0,0xf1b532b5de679c2498841c0aecc88d292a224869bcd9767e786d0e8e4eb1b820?tab=block)

让我们仔细看看这笔可疑交易: 它是一笔向自己转账且不包含任何数据的交易。它还支付了 1,715.244527903 Gwei 的 gas 费用。

<br>

![](images/eigenphi_7.png)
[BscScan 上的第0位交易](https://bscscan.com/tx/0x57159629b44d1f2e9383b1f21ef7a223696026094146b25503b995fc6b41b308)

三明治攻击者通过第0位交易将利润从 $22.455143 减少到了 $2.155413。

提交的 gas 费用是支付给验证者的。三明治攻击者通过出价高于其他攻击者赢得了受害者交易的打包。交易0被用来支付 gas 费用以确保在 48 Club 的竞价过程中赢得胜利。

这可能会引发一个问题: 为什么我们要将支付费用的交易分开? 我们本可以在交易1中包含费用,但交易成本是按照 $\frac{Gas Usage \times Gas Price}{10^9} \text{BNB}$ 计算的。虽然可以用 $GasPrice$ 来控制费用,但很难快速准确地计算 $Gas Usage$。这就是为什么我们在交易0中固定 $GasUsage=21,000$ 并用 $GasPrice$ 来控制费用。

## 常见问题

#### 为什么选择 BSC 网络?
以太坊网络的费用可能过高,这促使我选择了 gas 固定为 1 Gwei 的 BSC 网络。

你还可以使用 48 Club 的 Soul Point 功能以 0 Gwei 执行交易来进一步优化交易成本。

#### 你未来是否计划继续开发套利或 MEV?
由于资金不足,我无法继续运营。我确定需要价值 10 万至 100 万美元的代币才能从三明治攻击中获得净利润。

<br>
<br>
<br>

## 更多信息
- 页面
  - [Flashbots 主页](https://www.flashbots.net)
  - [MEV: 前五年](https://medium.com/@Prestwich/mev-c417d9a5eb3d)
  - [Vitalik Buterin 撰写的改进 x*y=k 做市商的前置运行抵抗性](https://ethresear.ch/t/improving-front-running-resistance-of-x-y-k-market-makers/1281)
  - [TxBoost](http://txboost.com)
- GitHub BSC 仓库问题
  - 关于 bloXroute
    - [bloxroute 利用用户进行 MEV](https://github.com/bnb-chain/bsc/issues/1706)
    - [bloxroute 继续破坏 BSC](https://github.com/bnb-chain/bsc/issues/1871)
    - [1Gwei 但区块中位置为 0 !!!](https://github.com/bnb-chain/bsc/issues/1831)
    - [BloxRoute 提供的 mev 服务中的交易位置不遵守官方 gwei 排序](https://github.com/bnb-chain/bsc/issues/1728)
  - 关于 gasPrice
    - [0 gwei 的交易又回来了](https://github.com/bnb-chain/bsc/issues/1746)
    - [0 gwei 的交易又回来了²](https://github.com/bnb-chain/bsc/issues/2371)

<br>
<br>
<br>

## Star 历史

<a href="https://star-history.com/#DonggeunYu/MEV-Attack-on-the-BSC&Date">
 <picture>
   <source media="(prefers-color-scheme: dark)" srcset="https://api.star-history.com/svg?repos=DonggeunYu/MEV-Attack-on-the-BSC&type=Date&theme=dark" />
   <source media="(prefers-color-scheme: light)" srcset="https://api.star-history.com/svg?repos=DonggeunYu/MEV-Attack-on-the-BSC&type=Date" />
   <img alt="Star History Chart" src="https://api.star-history.com/svg?repos=DonggeunYu/MEV-Attack-on-the-BSC&type=Date" />
 </picture>
</a>

### 解释 Python 代码

#### 重要说明
- 我们使用 `pyrevm` 来模拟交易。这使我们能够在不实际提交交易的情况下验证交易是否会成功。
- 我们使用 `multicall` 来批量获取池的状态。这减少了 RPC 调用的次数。
- 我们使用 `web3` 来与区块链交互。
- 我们使用 `asyncio` 来处理异步操作。

#### 书面过程说明
1. 监听内存池中的交易
   - 使用 `web3.eth.filter('pending')` 来监听新的待处理交易。
   - 使用 `web3.eth.get_transaction()` 来获取交易详情。

2. 分析交易
   - 检查交易是否调用了 DEX 合约。
   - 使用 `pyrevm` 模拟交易以获取预期的价格影响。
   - 计算可能的利润。

3. 执行攻击
   - 如果发现盈利机会,构建前置和后置交易。
   - 使用 `web3.eth.send_raw_transaction()` 提交交易。
   - 对于 MEV-Relay 路径,使用其 API 提交打包。

#### 为什么套利只用于普通路径?
套利交易通常需要更快的执行速度,而 MEV-Relay 会增加延迟。此外,套利的利润通常较小,支付 MEV-Relay 费用可能会抵消利润。

#### 追踪交易
我们使用以下方法来追踪交易:

1. 交易哈希追踪
   - 使用 `web3.eth.wait_for_transaction_receipt()` 等待交易被确认。
   - 检查交易收据以确认交易是否成功。

2. 错误处理
   - 如果交易失败,分析失败原因。
   - 记录错误信息以供后续分析。

3. 利润计算
   - 计算实际获得的利润。
   - 考虑 gas 费用和其他成本。

4. 日志记录
   - 记录所有重要信息,包括:
     * 交易哈希
     * 利润/损失
     * gas 费用
     * 错误信息(如果有)