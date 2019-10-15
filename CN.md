---
编号: "0020"
类别: 资料
状态: 草案
作者: <TBD>
组织: Nervos基金会
创建: 2019-6-19
---
# CKB Consensus Protocol
 
 # CKB 共识协议
  
  ![文档结构图](CKB%20%E5%85%B1%E8%AF%86%E5%8D%8F%E8%AE%AE.png)
  
* [概述](#Abstract)
* [研究动机](#Motivation)
* [技术概述](#Technical-Overview)
  * [消除区块传播瓶颈](#Eliminating-the-Bottleneck-in-Block-Propagation)
  * [利用缩短的延迟提高吞吐量](#Utilizing-the-Shortened-Latency-for-Higher-Throughput)
  * [避免自私挖矿攻击](#Mitigating-Selfish-Mining-Attacks)
* [规范](#Specification)
  * [两步交易确认](#Two-Step-Transaction-Confirmation)
  * [动态难度调节机制](#Dynamic-Difficulty-Adjustment-Mechanism)
 

<a name="Abstract"></a>
## 摘要

Bitcoin's Nakamoto Consensus (NC) is well-received due to its simplicity and low communication overhead. However, NC suffers from two kinds of drawback: first, its transaction processing throughput is far from satisfactory; second, it is vulnerable to a selfish mining attack, where attackers can gain more block rewards by deviating from the protocol's prescribed behavior.

比特币的中本聪共识（Nakamoto Consensus，简称 NC） 因其简单性和低通信开销的特点而广受好评。然而，NC 有两大缺陷：第一，其交易吞吐量远远无法满足现实需求；第二，NC 容易遭受自私挖矿攻击，在这一类攻击中，攻击者能够采用非协议规定的行为获得更多的出块奖励。

The CKB consensus protocol is a variant of NC that raises its performance limit and selfish mining resistance while keeping its merits. By identifying and eliminating the bottleneck in NC's block propagation latency, our protocol supports very short block interval without sacrificing security. The shortened block interval not only raises the throughput, but also lowers the transaction confirmation latency. By incorporating all valid blocks in the difficulty adjustment, selfish mining is no longer profitable in our protocol.

CKB 的共识协议是 NC 的变体，在保留 NC 优点的同时，提升了其性能极限和抵抗自私挖矿攻击的能力。通过研究发现消除 NC 区块广播延时的瓶颈，我们的协议能够在不牺牲安全性的情况下，支持非常短的出块间隔。缩短的出块间隔不仅提高了吞吐量，也降低了交易确认时长。通过在难度调节时考虑所有有效区块，在我们的协议中自私挖矿将不再有利可图。

<a name="Motivation"></a>
## 动机

Although a number of non-NC consensus mechanisms have been proposed, NC has the following threefold advantage comparing with its alternatives. First, its security is carefully scrutinized and well-understood [[1](https://www.cs.cornell.edu/~ie53/publications/btcProcFC.pdf), [2](https://eprint.iacr.org/2014/765.pdf), [3](https://fc16.ifca.ai/preproceedings/30_Sapirshtein.pdf), [4](https://eprint.iacr.org/2016/454.pdf), [5](https://eprint.iacr.org/2016/1048.pdf), [6](https://eprint.iacr.org/2018/800.pdf), [7](https://eprint.iacr.org/2018/129.pdf), [8](https://arxiv.org/abs/1607.02420)], whereas alternative protocols often open new attack vectors, either unintentionally [[1](http://fc19.ifca.ai/preproceedings/180-preproceedings.pdf), [2](https://www.esat.kuleuven.be/cosic/publications/article-3005.pdf)] or by relying on security assumptions that are difficult to realize in practice [[1](https://arxiv.org/abs/1711.03936), [2](https://arxiv.org/abs/1809.06528)]. Second, NC minimizes the consensus protocol's communication overhead. In the best-case scenario, propagating a 1MB block in Bitcoin is equivalent to broadcasting a compact block message of roughly 13KB [[1](https://github.com/bitcoin/bips/blob/master/bip-0152.mediawiki), [2](https://www.youtube.com/watch?v=EHIuuKCm53o)]; valid blocks are immediately accepted by all honest nodes. In contrast, alternative protocols often demand a non-negligible communication overhead to certify that certain nodes witness a block. For example, [Algorand](https://algorandcom.cdn.prismic.io/algorandcom%2Fa26acb80-b80c-46ff-a1ab-a8121f74f3a3_p51-gilad.pdf) demands that each block be accompanied by 300KB of block certificate. Third, NC's chain-based topology ensures that a transaction global order is determined at block generation, which is compatible with all smart contract programming models. Protocols adopting other topologies either [abandon the global order](https://allquantor.at/blockchainbib/pdf/sompolinsky2016spectre.pdf) or establish it after a long confirmation delay [[1](https://eprint.iacr.org/2018/104.pdf), [2](https://eprint.iacr.org/2017/300.pdf)], limiting their efficiency or functionality.

尽管目前市面上已经提出了许多非 NC 共识机制，但 NC 和这些替代方案相比，有以下三重优势：首先，它的安全性已经经过仔细地审查并被广泛理解[[1](https://www.cs.cornell.edu/~ie53/publications/btcProcFC.pdf), [2](https://eprint.iacr.org/2014/765.pdf), [3](https://fc16.ifca.ai/preproceedings/30_Sapirshtein.pdf), [4](https://eprint.iacr.org/2016/454.pdf), [5](https://eprint.iacr.org/2016/1048.pdf), [6](https://eprint.iacr.org/2018/800.pdf), [7](https://eprint.iacr.org/2018/129.pdf), [8](https://arxiv.org/abs/1607.02420)]，但其它的替代协议常常引入了新的攻击维度，或无意[1](http://fc19.ifca.ai/preproceedings/180-preproceedings.pdf), [2](https://www.esat.kuleuven.be/cosic/publications/article-3005.pdf)] 或依赖于实践中难以达到的安全性前提[[1](https://arxiv.org/abs/1711.03936), [2](https://arxiv.org/abs/1809.06528)]。第二，NC 最大程度地降低了共识协议的通信开销。最理想的情况下，在比特币网络中广播 1MB 的区块和广播一个大约 13KB 的致密区块信息是等价的[[1](https://github.com/bitcoin/bips/blob/master/bip-0152.mediawiki), [2](https://www.youtube.com/watch?v=EHIuuKCm53o)]；有效区块会立刻被所有诚实节点接受。相比之下，其它替代性协议通常需要不可忽视的通信开销来确认一个节点见证了一个区块。举例来说，[Algorand](https://algorandcom.cdn.prismic.io/algorandcom%2Fa26acb80-b80c-46ff-a1ab-a8121f74f3a3_p51-gilad.pdf)要求每一个区块要附带一个 300KB 的区块认证。第三，NC 基于链的拓扑结构确保了其在区块生成的时候，就能确认全局交易排序，这和所有智能合约编程模型兼容。采用其它拓扑结构的协议要么放弃了[全局交易排序](https://allquantor.at/blockchainbib/pdf/sompolinsky2016spectre.pdf)，要么在很长的确认延迟之后才对交易进行排序[[1](https://eprint.iacr.org/2018/104.pdf), [2](https://eprint.iacr.org/2017/300.pdf)]，这样会限制它们的效率或者性能。

Despite NC's merits, a scalability barrier hinders it from processing more than a few transactions per second. Two parameters collectively cap the system's throughput: the maximum block size and the expected block interval. For example, Bitcoin enforces a roughly 4MB block size upper bound and targets a 10-minute block interval and  with its **difficulty adjustment mechanism**, translating to roughly ten transactions per second (TPS). Increasing the block size or reducing the block interval leads to longer block propagation latency or more frequent block generation events, respectively; both approaches raise the fraction of blocks generated during other blocks' propagation, thus raising the fraction of competing blocks. As at most one block among the competing ones contributes to transaction confirmation, the nodes' bandwidth on propagating other **orphaned blocks** is wasted, limiting the system's effective throughput. Moreover, raising the orphan rate downgrades the protocol's security by lowering the difficulty of double-spending attacks [[1](<https://fc15.ifca.ai/preproceedings/paper_30.pdf>), [2](<https://fc15.ifca.ai/preproceedings/paper_101.pdf>)].

尽管 NC 有很多优点，但其可扩展性的问题使得它每秒仅能处理少数交易。采用 NC 协议的网络，其吞吐量会受到两个参数的共同限制：1）最大区块容量，2）预期出块间隔。举例来说，比特币的区块大小上限约为 4MB，通过**难度调节机制**，其目标出块间隔被设定为大约 10 分钟，换算过来吞吐量（TPS）大约为每秒 10 笔交易。提高区块容量或缩短出块间隔分别会导致更长的区块广播延时或生成更多的区块。这两种方法都会提升在其它区块广播过程中，区块生成的概率，从而会增大竞争块出现的概率。由于在诸多竞争块之间，最多只有一个区块能够为交易确认作出贡献，因此浪费了广播其他**孤块**的节点带宽，从而限制了系统的有效吞吐量。此外，孤块率的提高会降低双花攻击的难度，这会导致系统的安全性降低[[1](<https://fc15.ifca.ai/preproceedings/paper_30.pdf>), [2](<https://fc15.ifca.ai/preproceedings/paper_101.pdf>)]，从而降低协议的安全性。

Moreover, the security of NC is undermined by a [**selfish mining attack**](https://www.cs.cornell.edu/~ie53/publications/btcProcFC.pdf), which allows attackers to gain unfair block rewards by deliberately orphaning blocks mined by other miners. Researchers observe that the unfair profit roots in NC's difficulty adjustment mechanism, which neglects orphaned blocks when estimating the network's computing power. Through this mechanism, the increased orphan rate caused by selfish mining leads to lower mining difficulty, enabling the attacker's higher time-averaged block reward [[1](https://eprint.iacr.org/2016/555.pdf), [2](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-100.md), [3](https://arxiv.org/abs/1805.08281)].

另外，NC 的安全性也会遭到[**自私挖矿攻击**](https://www.cs.cornell.edu/~ie53/publications/btcProcFC.pdf)的破坏，自私挖矿攻击者可以通过故意地孤立其它矿工的区块，来获得不合理的区块奖励。研究人员发现，这种不公平盈利机会的根源在于 NC 的难度调整机制，它在估算网络计算能力时忽略了孤块。通过这种机制设计，自私挖矿攻击会提升孤块率，从而导致挖矿难度降低，这会让攻击者在单位时间内获得高于平均值的区块奖励[[1](https://eprint.iacr.org/2016/555.pdf), [2](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-100.md), [3](https://arxiv.org/abs/1805.08281)]。

In this RFC, we present the CKB consensus protocol, a consensus protocol that raises NC's performance limit and selfish mining resistance while keeping all NC's merits. Our protocol supports very short block interval by reducing the block propagation latency. The shortened block interval not only raises the blockchain's throughput, but also minimizes the transaction confirmation latency without decreasing the level of confidence, as the orphan rate remains low. Selfish mining is no longer profitable as we incorporate all blocks, including uncles, in the difficulty adjustment when estimating the network's computing power, so that the new difficulty is independent of the orphan rate.

在这个 RFC 中，我们提出了 CKB 的共识协议，它能够提升 NC 的性能限制和抗自私挖矿能力，同时保留 NC 所有的优点。我们的协议通过降低区块广播延时来支持非常短的出块间隔。更短的出块间隔不仅可以增加吞吐量，还能够在不牺牲安全性的前提下，降低交易确认延迟（因为孤块率保持在较低的水平）。由于在估计全网算力时，我们会将所有区块（包括叔块）纳入到难度调节中，因此新的难度和孤块率是不相关的，这让自私挖矿不再有利可图。

<a name="Technical-Overview"></a>
## 技术概述

Our consensus protocol makes three changes to NC.

我们的共识协议在 NC 的基础上做了三个改进。

<a name="#Eliminating-the-Bottleneck-in-Block-Propagation"></a>
### 消除区块传播瓶颈

[Bitcoin's developers identify](https://www.youtube.com/watch?v=EHIuuKCm53o) that when the block interval decreases, the bottleneck in block propagation latency is transferring **fresh transactions**, which are newly broadcast transactions that have not finished propagating to the network when embedded in the latest block. Nodes that have not received these transactions must request them before forwarding the block to their neighbors. The resulted delay not only limits the blockchain's performance, but can also be exploited in a **de facto selfish mining attack**, where attackers deliberately embed fresh transactions in their blocks, hoping that the longer propagation latency gives them an advantage in finding the next block to gain more rewards.


[比特币协议的开发人员发现](https://www.youtube.com/watch?v=EHIuuKCm53o)，当出块间隔降低的时候，区块传播延时的瓶颈是**新交易的传播**。新交易是指在最新的区块中包含的、尚未被传播到网络中的交易。没有收到过这些交易的节点必须要在向相邻节点广播收到的这个区块之前，请求这些新交易。由此产生的延迟不仅限制了区块链的性能，而且还会在**实际自私挖矿攻击**中被利用。在这类攻击中，攻击者会故意将新的交易并入在他们自己的区块中，这会为他们带来更长的广播延时，因此，他们能够在寻找下一个区块的竞争中取得一定优势，从而获得更多奖励。

Departing from this observation, our protocol eliminates the bottleneck by decoupling NC's transaction confirmation into two separate steps: **propose** and **commit**. A transaction is proposed if its truncated hash, named `txpid`, is embedded in the **proposal zone** of a blockchain block or its **uncles**---orphaned blocks that are referred to by the blockchain block. Newly proposed transactions affect neither the block validity nor the block propagation, as a node can start transferring the block to its neighbors before receiving these transactions. The transaction is committed if it appears in the **commitment zone** in a window starting several blocks after its proposal. This two-step confirmation rule eliminates the block propagation bottleneck, as committed transactions in a new block are already received and verified by all nodes when they are proposed. The new rule also effectively mitigates de facto selfish mining by limiting the attack time window.

从这个研究结果出发，我们的共识协议通过将 NC 的交易确认分解为两个步骤来消除区块传播瓶颈：1）**交易提交（Propose）**，2）**交易确认（Commit）**。如果一个交易的短哈希（称为 `txpid`）出现在一个区块或其中一个**叔块**（被区块引用的孤块）的「**交易提交区**」（Proposal Zone），则该交易被提交。新提交的交易既不会影响区块有效性，也不会影响区块的广播，因为一个节点在收到这些交易之前，就可以开始向相邻节点传播包含这些被提出的交易区块。在交易被提交之后需要经过数个区块的时间窗口，如果这个交易出现在区块的「**交易确认区**」（Commitment Zone）中，则该交易被打包。这样的两步交易确认机制消除了区块传播瓶颈，因为一个新块的确认交易在被提出时就已经被所有节点接收和验证。通过限制攻击时间窗口，新的机制也有效地减少了实际自私挖矿攻击。

<a name="Utilizing-the-Shortened-Latency-for-Higher-Throughput"></a>
### 利用缩短的延迟提高吞吐量

Our protocol prescribes that blockchain blocks refer to all orphaned blocks as uncles. This information allows us to estimate the current block propagation latency and dynamically adjust the expected block interval, increasing the throughput when the latency improves. Accordingly, our difficulty adjustment targets a fixed orphan rate to utilize the shortened latency without compromising security. The protocol hard-codes the upper and lower bounds of the interval to defend against DoS attacks and avoid overloading the nodes. In addition, the block reward is adjusted proportionally to the expected block interval within an epoch, so that the expected time-averaged reward is independent of the block interval.

我们的协议将区块链中所有引用孤块的区块称为叔块。这让我们能够很好地估算当前区块传播的延时，并动态地调节期望的出块间隔，从而在延时得到改善时，提升吞吐量。因此，我们设定固定的孤块率作为难度调节目标，它让我们在不牺牲安全性的情况下，能够利用更短的延迟。协议中硬编码了出块间隔的上下限来防止 DoS 攻击，和避免节点超载。此外，在每个难度调节周期（Epoch）中，区块奖励会根据预期出块间隔，成比例地进行调节，因此预期的平均时间奖励和出块间隔并不相关。

<a name="Mitigating-Selfish-Mining-Attacks"></a>
### 减少自私挖矿攻击

Our protocol incorporate all blocks, including uncles, in the difficulty adjustment when estimating the network's computing power, so that the new difficulty is independent of the orphan rate, following the suggestion of [Vitalik](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-100.md), [Grunspan and Perez-Marco](https://arxiv.org/abs/1805.08281).

根据[Vitalik](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-100.md), [Grunspan 和 Perez-Marco](https://arxiv.org/abs/1805.08281).的建议，我们的协议在估计网络算力时，会将所有区块（包括叔块）纳入到难度调节中，因此新的难度和孤块率是不相关的。

In addition, we prove that selfish mining is no longer profitable in our protocol. This prove is non-trivial as Vitalik, Grunspan and Perez-Marco's informal arguments do not rule out the possibility that the attacker adapts to the modified mechanism and still gets unfair block reward. For example, the attacker may temporarily turn off some mining gears in the first epoch, causing the modified difficulty adjustment algorithm to underestimate the network's computing power, and starts selfish mining in the second epoch for a higher overall time-averaged reward. We prove that in our protocol, selfish mining is not profitable regardless of how the attacker divides its mining power among honest mining, selfish mining and idle, and how many epochs the attack involves. The detailed proof will be released later.

此外，我们证明了在我们的协议中，自私挖矿不再有利可图。这个证明是非常有意义的，因为在 Vitalik，Grunspan 和 Perez-Marco 的非正式论证中，并没有排除攻击者在新的协议机制下，仍然获得不公平奖励的可能性，而我们的协议证明了这一点。举例来说，攻击者可能会在第一个难度调节周期中暂时关闭一些矿机，这样就会导致修改后的难度调节算法会低估全网算力，在第二个难度调节周期时，这些攻击者便开始自私挖矿，以在整体上获得更高的平均时间收益。我们证明了在我们的协议中，不论攻击者在诚实挖矿、自私挖矿、关闭矿机这三种策略中如何分配算力，不论他的策略覆盖多少个难度调节周期，自私挖矿都是无利可图的。证明的细节我们会在接下来的部分公布。

<a name="Specification"></a>
## 规范

<a name="Two-Step-Transaction-Confirmation"></a>
### 两步交易确认

In our protocol, we use a two-step transaction confirmation to eliminate the aforementioned block propagation bottleneck, regardless of how short the block interval is. We start by defining the two steps and the block structure, and then introduce the new block propagation protocol. 

在我们的协议中，我们采用了两步交易确认机制来消除上述的区块广播瓶颈（无论出块间隔多短）。我们首先来定义一下这两个步骤和区块结构，之后会介绍新的区块传播协议。

#### 定义

> **Definition 1:** A transaction’s proposal id `txpid` is defined as the first *l* bits of the transaction hash `txid`.

In our protocol, `txpid` does not need to be as globally unique as `txid`, as a `txpid` is used to identify a transaction among several neighboring blocks. Since we embed `txpid`s in both blocks and compact blocks, sending only the truncated `txid`s could reduce the bandwidth consumption. 

When multiple transactions share the same `txpid`s, all of them are considered proposed. In practice, we can set *l* to be large enough so that the computational effort of finding a collision is non-trivial.


> **定义 1:** 一笔交易的提交 id `txpid` 被定义为这笔交易哈希 `txid` 的前 *l* 位比特。

`txpid` 用于标识几个相邻区块之间的交易，所以在我们的协议中，`txpid` 不需要和 `txid` 那样具有全局唯一性。另外，由于我们在区块和致密区块中都嵌入了 `txpid`，因此只发送缩短的 `txid` 可以减少带宽消耗。

当出现多笔交易使用相同的 `txpid` 时，那么所有的这些交易都会被认为是已经提交的。实际上，我们可以将 *l* 设置得足够大，这样查找碰撞所需的计算量会是较为合理的。

> **Definition 2:** A block *B*<sub>1</sub> is considered to be the *uncle* of another block *B*<sub>2</sub> if all of the following conditions are met:
>​	(1) *B*<sub>1</sub> and *B*<sub>2</sub> are in the same epoch, sharing the same difficulty;
>​	(2) height(*B*<sub>2</sub>) > height(*B*<sub>1</sub>);
>​	(3) *B*<sub>2</sub> is the first block in its chain to refer to *B*<sub>1</sub>. 

Our uncle definition is different from [that of Ethereum](https://github.com/ethereum/wiki/wiki/White-Paper#modified-ghost-implementation), in that we do not consider how far away the two blocks' first common ancestor is, as long as the two blocks are in the same epoch.

> **定义 2:** 当满足所有下述条件时，区块 *B*<sub>1</sub> 就被认为是另一个区块 *B*<sub>2</sub> 的叔块：
>​	(1) 在同一个难度调节周期中，*B*<sub>1</sub> 和 *B*<sub>2</sub> 有相同的出块难度；
>​	(2) *B*<sub>2</sub> 的区块高度大于 *B*<sub>1</sub> 的区块高度；
>​	(3) *B*<sub>2</sub> 是在链上第一个引用 *B*<sub>1</sub> 的区块。

我们对叔块的定义不同于[以太坊](https://github.com/ethereum/wiki/wiki/White-Paper#modified-ghost-implementation)，我们不考虑两个区块距离它们第一个共同的起始区块有多远，只要两个区块在同一个难度调节周期即可。

> **Definition 3:** A transaction is *proposed* at height *h*<sub>p</sub> if its `txpid` is in the proposal zone of the main chain block with height *h*<sub>p</sub> and this block’s uncles. 

It is possible that a proposed transaction is previously proposed, in conflict with other transactions, or even malformed. These incidents do not affect the block’s validity, as the proposal zone is used to facilitate transaction synchronization.

> **定义 3:** 如果一笔交易的 `txpid` 被包含在区块高度为 *h*<sub>p</sub> 的主链区块（或者该区块的叔块）的提交区中，则这笔交易在高度 *h*<sub>p</sub> 被提交。

这里有可能会出现这样的情况，一笔提案的交易可能是以前提出的，这与其它交易冲突，甚至出现异常。由于交易提交区只是用来促成交易同步，因此这些情况不会影响区块的有效性。

> **Definition 4:** A non-coinbase transaction is *committed* at height *h*<sub>c</sub> if all of the following conditions are met: 
> ​	(1) the transaction is proposed at height *h*<sub>p</sub> of the same chain, and *w<sub>close</sub>  ≤  h<sub>c</sub> − h*<sub>p</sub>  ≤  *w<sub>far</sub>*
> ​	(2) the transaction is in the commitment zone of the main chain block with height *h*<sub>c</sub>; 
> ​	(3) the transaction is not in conflict with any previously-committed transactions in the main chain. 
> The coinbase transaction is committed at height *h*<sub>c</sub> if it satisfies (2).

*w<sub>close</sub>* and *w<sub>far</sub>* define the closest and farthest on-chain distance between a transaction’s proposal and commitment. We require *w<sub>close</sub>*  to be large enough so that *w<sub>close</sub>* block intervals are long enough for a transaction to be propagated to the network. 

These two parameters are also set according to the maximum number of transactions in the proposed transaction pool of a node’s memory. As the total number of proposed transactions is limited, they can be stored in the memory so that there is no need to fetch a newly committed transaction from the hard disk in most occasions. 

A transaction is considered embedded in the blockchain when it is committed. Therefore, a receiver that requires σ confirmations needs to wait for at least *w<sub>close</sub>* +σ blocks after the transaction is broadcast to have confidence in the transaction. 

In practice, this *w<sub>close</sub>* - block extra delay is compensated by our protocol’s shortened block interval, so that the usability is not affected.

> **定义 4:** 当满足所有下述条件时，则一笔非币基交易在区块高度 *h*<sub>c</sub> 上被确认：

一个非coinbase交易将被提交在高度 *h*<sub>c</sub> ，如果满足以下所有条件
> ​	(1) 该交易在同一条链上的区块高度 *h*<sub>p</sub> 上被提交，并且 *w<sub>close</sub>  ≤  h<sub>c</sub> − h*<sub>p</sub>  ≤  *w<sub>far</sub>* ；
> ​	(2) 该交易位于区块高度为  *h*<sub>c</sub> 的主链区块交易确认区；
> ​	(3) 该交易没有和主链上任何之前确认的交易冲突。

> 如果是币基交易，那么满足条件（2）就会在区块高度 *h*<sub>c</sub> 被确认。

*w<sub>close</sub>* 和 *w<sub>far</sub>* 分别定义了交易提交和确认的最近及最远的链上距离。我们要求 *w<sub>close</sub>* 足够大，这样才能使得 *w<sub>close</sub>* 的区块间隔足够长，以便将交易广播到整个网络。

这两个参数还会根据节点内存中交易提案池能够保存的最大交易数量来设定。由于提交交易的总量有限，所以可以将它们保存在内存中，因此在大多数情况下不需要从硬盘中抓取最新提交的交易。

只有在交易确认之后，一笔交易才算被记录在区块链中。因此，在要求 σ 个区块确认的情况下，一个接收者在区块广播之后需要等待至少 *w<sub>close</sub>* + σ 个区块，才能确保交易有效。

在实践中，由于我们的协议有更短的出块间隔，*w<sub>close</sub>* 时间间隔带来的额外延迟被抵消，因此实用性并不会受到影响。

#### 区块和致密区块结构

在我们的协议中，一个区块包含以下几个字段:

| 名称            | 描述                                 |
| :-------------- | :----------------------------------- |
| 区块头           | 包含区块元数据                       |
| 确认区           | 此区块中确认的交易             |
| 提交区     | 在此区块中提交的 `txpid`    |
| 叔块头   | 叔块的区块头             |
| 叔块的提交区   | 在叔块中提交的 `txpid`               |

Similar to NC, in our protocol, a compact block replaces a block’s commitment zone with the transactions’ `shortid`s, a salt and a list of prefilled transactions. All other fields remain unchanged in [the compact block](https://github.com/bitcoin/bips/blob/master/bip-0152.mediawiki).

与 NC 类似，在我们的协议中，致密区块会使用交易的 `shortid` 替代区块的确认区，里面是预先提交的交易列表和 salt。[致密区块](https://github.com/bitcoin/bips/blob/master/bip-0152.mediawiki)的其它所有字段保持不变。

Additional block structure rules:

附加区块结构规则：

- The total size of the first four fields should be no larger than the hard-coded **block size limit**. The main purpose of implementing a block size limit is to avoid overloading public nodes' bandwidth. The uncle blocks’ proposal zones do not count in the limit as they are usually already synchronized when the block is mined. 
- The number of `txpid`s in a proposal zone also has a hard-coded upper bound.

- 前四个字段的总大小不能大于硬编码的**区块大小限制**。设定区块大小限制的主要目的是避免公共节点带宽过载。由于叔块的提交区通常在出块时就已经被同步，所以它们不会被计入在区块容量限制内。

- 提交区中`txpid` 的数量也会有硬编码的上限。

Two heuristic requirements may help practitioners choose the parameters. First, the upper bound number of `txpid`s in a proposal zone should be no smaller than the maximum number of committed transactions in a block, so that even if *w<sub>close</sub>=w<sub>far</sub>*, this bound is not the protocol's throughput bottleneck. Second, ideally the compact block should be no bigger than 80KB. According to [a 2016 study by Croman et al.](https://fc16.ifca.ai/bitcoin/papers/CDE+16.pdf), messages no larger than 80KB have similar propagation latency in the Bitcoin network; larger messages propagate slower as the network throughput becomes the bottleneck. This number may change as the network condition improves.

两个启发式的要求能够帮助实践者选择参数。首先，在交易提交区 `txpid` 的上限不会小于一个区块中确认交易的最大数量，因此即使 *w<sub>close</sub>=w<sub>far</sub>* ，这个限制也不会成为协议吞吐量的瓶颈。其次，理想情况下，致密区块的大小不会超过 80KB。[Croman 等人 在 2016 年的研究](https://fc16.ifca.ai/bitcoin/papers/CDE+16.pdf)表明，不大于 80KB 的消息在比特币网络中的广播延迟较为相似；更大信息的广播速度会因为网络吞吐量这个瓶颈而变慢。当然，这个数字会随着网络条件的提升而改变。

#### 区块传播协议

In line with [[1](https://www.cs.cornell.edu/~ie53/publications/btcProcFC.pdf), [2](https://arxiv.org/abs/1312.7013), [3](https://eprint.iacr.org/2014/007.pdf)], nodes should broadcast all blocks with valid proofs-of-work, including orphans, as they may be referred to in the main chain as uncles. Valid proofs-of-work cannot be utilized to pollute the network, as constructing them is time-consuming. 

根据[[1](https://www.cs.cornell.edu/~ie53/publications/btcProcFC.pdf), [2](https://arxiv.org/abs/1312.7013), [3](https://eprint.iacr.org/2014/007.pdf)], 所述，节点应该广播所有带有有效工作量证明的区块，包括孤块，它们可能是被主链引用的叔块。由于构建有效的工作量证明会消耗大量时间，因此这些证明不会用于败坏网络。

Our protocol’s block propagation protocol removes the extra round trip of fresh transactions in most occasions. When the round trip is inevitable, our protocol ensures that it only lasts for one hop in the propagation. This is achieved by the following three rules: 

我们协议的区块传播协议在大多数情况下能够消除新交易的额外一轮广播。当它不可避免时，我们的协议会确保新交易在往返广播中仅持续一次反射。这是通过以下三个规则实现的：

1. If some committed transactions are previously unknown to the sending node, they will be embedded in the prefilled transaction list and sent along with the compact block. This only happens in a de facto selfish mining attack, as otherwise transactions are synchronized when they are proposed. This modification removes the extra round trip if the sender and the receiver share the same list of proposed, but-not-broadcast transactions. 

1. 如果一些已经确认的交易对发送的节点来说是未知的，他们将会被放在预先填写的交易列表和致密区块中一同发送。这只会在实际自私挖矿攻击中发生，在其它情况下交易会在提交的时候就被同步。如果接收方和发送方共享同一份提交列表，但是没有广播的交易列表，这项改进就能够移除额外的一轮通信。

2. If certain committed transactions are still missing, the receiver queries the sender with a short timeout. Triggering this mechanism requires not only a successful de facto selfish mining attack, but also an attack on transaction propagation to cause inconsistent proposed transaction pools among the nodes. Failing to send these transactions in time leads to the receiver disconnecting and blacklisting the sender. Blocks with incomplete commitment zones will not be propagated further.

2. 如果某项确认的交易仍然是缺失的，那么接收者就会向发送者请求交易，并且发送者必须在很短的时间内返回交易。触发此机制不仅要求一次成功的实际自私挖矿攻击，还需要对交易广播进行攻击，以使节点之间产生不一致的提案交易池。若未能及时发送这些交易，会导致接收方断开链接并将发送方列入到黑名单。带有不完整确认区的区块将不会被进一步广播。

3. As long as the commitment zone is complete and valid, a node can start forwarding the compact block before receiving all newly-proposed transactions. In our protocol, a node requests the newly-proposed transactions from the upstream peer and sends compact blocks to other peers simultaneously. This modification does not downgrade the security as transactions in the proposal zone do not affect the block’s validity.

3. 一旦确认区是完整并且有效的，节点就能够在接收所有新提交的交易之前，开始传播致密区块。在我们的协议中，一个节点需要从上游节点请求新提交的交易，同时将致密区块发送给其它节点。由于提交区的交易不会影响区块的有效性，这项改进不会降低安全性。

The first two rules ensure that the extra round trip caused by a de facto selfish mining attack never lasts for more than one hop.

前两条规则保证了因实际自私挖矿攻击造成的额外通信持续时间，不会超过一次反射。

<a name="Dynamic-Difficulty-Adjustment-Mechanism"></a>
### 动态难度调整机制

We modify the Nakamoto Consensus difficulty adjustment mechanism, so that: (1) Selfish mining is no longer profitable; (2) Throughput is dynamically adjusted based on the network’s bandwidth and latency. To achieve (1), our protocol incorporates all blocks, instead of only the main chain, in calculating the adjusted hash rate estimation of the last epoch, which determines the amount of computing effort required in the next epoch for each reward unit. To achieve (2), our protocol calculates the number of main chain blocks in the next epoch with the last epoch’s orphan rate. The block reward and target are then computed by combining these results.

我们写改了中本聪共识的难度调节机制，因此：（1）自私挖矿不再有利可图；（2）吞吐量会根据网络带宽和延迟动态地进行调整。为了实现（1），我们的协议在每个难度调节周期估计全网哈希率的时候，考虑了所有区块，以确定下一个难度调节周期每个奖励单元所需的计算工作量，而不仅仅是主链的区块。为了实现（2），我们的协议根据最新难度调节周期的孤块率来计算下一个难度调节周期中主链区块的数量。最后，结合这些结果来计算区块奖励和调节目标。

Additional constraints are introduced to maximize the protocol’s compatibility:

我们还引入了其它约束，来最大化协议的可兼容性：

1. All epochs have the same expected length Lideal, and the maximum block reward issued in an epoch R(i) depends only on the epoch number i, so that the dynamic block interval does not complicate the reward issuance policy.

1. 1、所有的难度调节周期都有相同的预期长度 *L<sub>ideal</sub>* ，并且在难度调节周期 R(*i*) 中发放的最大区块奖励仅取决于每个难度调节周期数 *i*，因此动态出块间隔并不会让奖励发放机制复杂化。

2. Several upper and lower bounds are applied to the hash rate estimation and the number of main chain blocks, so that our protocol does not harm the decentralization or attack-resistance of the network.

2. 我们会给主链区块数量和哈希率的预估设定多个上下限，因此我们的协议不会影响网络的去中心化以及抗攻击能力。

#### Notations
#### 注释

Similar to Nakamoto Consensus , our protocol’s difficulty adjustment algorithm is executed at the end of every epoch. It takes four inputs:

和中本聪共识类似，我们协议的难度调节算法在每个难度调节周期末执行。该算法需要四个入参：

| 名称            | 描述                          |
| :-------------- | :----------------------------------- |
| *T*<sub>*i*</sub>          | 上一个难度调节周期的目标                     |
| *L*<sub>*i*</sub> | 上一个难度调节周期的持续时间，即周期 *i* 和周期（*i-1*）的最后一个区块的时间戳之差 |
| *C*<sub>*i*,m</sub>   | 上一个难度调节周期主链区块总数      |
| *C*<sub>*i*,o</sub>   | 上一个难度调节周期的孤块总数：在难度调节周期 *i* 中主链包含的叔块总数        |

Among these inputs, *T<sub>i</sub>* and *C*<sub>*i*,m</sub> are determined by the last iteration of difficulty adjustment; *L*<sub>*i*</sub> and *C*<sub>*i*,o</sub> are measured after the epoch ends. The orphan rate *o*<sub>*i*</sub> is calculated as *C*<sub>*i*,o</sub> / *C*<sub>*i*,m</sub>. We do not include *C*<sub>*i*,o</sub> in the denominator to simplify the equation. As some orphans at the end of the epoch might be excluded from the main chain by an attack, *o*<sub>*i*</sub> is a lower bound of the actual number. However, [the proportion of deliberately excluded orphans is negligible](https://eprint.iacr.org/2014/765.pdf) as long as the epoch is long enough, as the difficulty of orphaning a chain grows exponentially with the chain length. 

在这些入参中，*T<sub>i</sub>* 和 *C*<sub>*i*,m</sub> 由上一个难度调节决定；*L*<sub>*i*</sub> 和 *C*<sub>*i*,o</sub> 在每个难度调节周期结束后取数。孤块率 *o*<sub>*i*</sub> 由  *C*<sub>*i*,o</sub> / *C*<sub>*i*,m</sub>得到。为了简化方程，我们不会将 *C*<sub>*i*,o</sub> 包含在分母中。由于在难度调节周期末期，一些孤块可能会被攻击从而不被包含在主链中，因此 *o*<sub>*i*</sub> 是实际数据的下限。然而，只要一个难度调节周期足够长，[被故意排除在外的孤块占比几乎可以忽略不计](https://eprint.iacr.org/2014/765.pdf)，因为孤立一条链的难度会随着链长度的增长呈指数级增长。

The algorithm outputs three values:

该算法会输出三个值：

| 名称            | 描述                          |
| :-------------- | :----------------------------------- |
| *T*<sub>*i*+1</sub>          | 下一个难度调节周期的目标                      |
| *C*<sub>i+1,m</sub> | 下一个难度调节周期的主链区块总数  |
| *r*<sub>*i*+1</sub>   | 下一个难度调节周期的区块奖励     |

If the network hash rate and block propagation latency remains constant, *o*<sub>*i*+1</sub> should reach the ideal value *o*<sub>ideal</sub>, unless *C*<sub>*i*+1,m</sub> is equal to its upper bound *C*<sub>m</sub><sup>max</sup>  or its lower bound *C*<sub>m</sub><sup>min</sup> . Epoch *i* + 1 ends when it reaches *C*<sub>*i*+1,m</sub> main chain blocks, regardless of how many uncles are embedded.

如果网络的哈希率和区块广播延时保持不变，*o*<sub>*i*+1</sub> 应该达到理想值  *o*<sub>ideal</sub>，除非 *C*<sub>*i*+1,m</sub>  等于其上限 *C*<sub>m</sub><sup>max</sup> 或它的下限 *C*<sub>m</sub><sup>min</sup>。难度调节周期 *i* + 1 在达到 *C*<sub>*i*+1,m</sub> 个主链区块时结束，无论其中包含了多少个叔块。

#### Computing the Adjusted Hash Rate Estimation

#### 计算调整后的哈希率估值

The adjusted hash rate estimation, denoted as *HPS<sub>i</sub>* is computed by applying a dampening factor τ to the last epoch’s actual hash rate ![1559068235154](images/1559068235154.png). The actual hash rate is calculated as follows:

调整后哈希率估值，表示为 *HPS<sub>i</sub>* ，是通过将阻尼因子 τ 作用于上一难度调节周期的实际哈希率 ![1559068235154](images/1559068235154.png) 计算。 实际哈希率计算如下:

![1559064934639](images/1559064934639.png)

where:

- HSpace is the size of the entire hash space, e.g., 2^256 in Bitcoin,
- HSpace/*T<sub>i</sub>* is the expected number of hash operations to find a valid block, and 
- *C*<sub>*i*,m</sub> + *C*<sub>*i*,o</sub> is the total number of blocks in epoch *i*

在这里：

- HSpace 是所有哈希空间的大小，如在比特币中就是 2²⁵⁶
- HSpace/*T<sub>i</sub>* 是找到一个有效区块预期所需要的哈希运算数量
- *C*<sub>*i*,m</sub> + *C*<sub>*i*,o</sub> 是难度调节周期 *i* 中所有的区块数量

![1559068266162](images/1559068266162.png) is computed by dividing the expected total hash operations with the duration *L<sub>i</sub>*

![1559068266162](images/1559068266162.png) 是通过预期总哈希运算量除以持续时长 *L<sub>i</sub>* 来计算的。

Now we apply the dampening filter:

现在我们引入过滤器（Dampening Filter）：

![1559064108898](images/1559064108898.png)

where *HPS*<sub>*i*−1</sub> denotes the adjusted hash rate estimation output by the last iteration of the difficulty adjustment algorithm. The dampening factor ensures that the adjusted hash rate estimation does not change more than a factor of τ between two consecutive epochs. This adjustment is equivalent to the Nakamoto Consensus application of a dampening filter. Bounding the adjustment speed prevents the attacker from arbitrarily biasing the difficulty and forging a blockchain, even if some victims’ network is temporarily controlled by the attacker.

在这里，*HPS*<sub>*i*−1</sub> 表示通过上一次难度调整算法的迭代，所得出的调整后哈希率估值输出。阻尼因子保证了在两个连续的难度调节周期中，调整后的哈希率估值变动不会多于 τ 因子。这种调整相当于中本聪共识过滤器的应用。为调节速度设定边界，可以防止了攻击者随意偏置难度并伪造区块链，即使一些受害者的网络被攻击者暂时控制。

#### Modeling Block Propagation

#### 区块传播建模

It is difficult, if not impossible, to model the detailed block propagation procedure, given that the network topology changes constantly over time. Luckily, for our purpose, it is adequate to express the influence of block propagation with two parameters, which will be used to compute *C*<sub>*i*+1,m</sub>  later.

由于网络的拓扑结构会随着时间不断变化，即使我们有可能为具体的区块广播过程建模，那也是非常困难的。幸运的是，对于我们的目标来说，用两个参数来表达区块广播的影响是完全足够的，这两个参数会在后面被用来计算 *C*<sub>*i*+1,m</sub> 。

We assume all blocks follow a similar propagation model, in line with [[1](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.395.8058&rep=rep1&type=pdf), [2](https://fc16.ifca.ai/bitcoin/papers/CDE+16.pdf)]. In the last epoch, it takes *d* seconds for a block to be propagated to the entire network, and during this process, the average fraction of mining power working on the block’s parent is *p*. Therefore, during this *d* seconds, *HPS*<sub>*i* </sub> × *dp* hash operations work on the parent, thus not contributing to extending the blockchain, while the rest *HPS*<sub>*i*</sub> × *d*(1 − *p*) hashes work on the new block. Consequently, in the last epoch, the total number of hashes that do not extend the blockchain is *HPS*<sub>*i*</sub>  × *dp* × *C*<sub>*i*,m</sub>. If some of these hashes lead to a block, one of the competing blocks will be orphaned. The number of hash operations working on observed orphaned blocks is HSpace/*T*<sub>*i*</sub> × *C*<sub>*i*,o</sub>. If we ignore the rare event that more than two competing blocks are found at the same height, we have:

我们假设所有区块都遵循类似的区块传播模型，符合[[1](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.395.8058&rep=rep1&type=pdf), [2](https://fc16.ifca.ai/bitcoin/papers/CDE+16.pdf)]。在上一个难度调节周期中，一个区块广播到整个网络需要花费 *d* 秒，这个过程中，在该区块的父节点上的平均挖矿算力占比为 *p*。因此，在这 *d* 秒期间，会有 *HPS*<sub>*i* </sub> × *dp* 的哈希运算花费在父节点上，而不会为增加区块链作出贡献，余下 *HPS*<sub>*i*</sub> × *d*(1 − *p*)  的哈希算力会花费在新区块上。结果就是，在上一个难度调节周期，没有作用于增长区块率的总哈希数为 *HPS*<sub>*i*</sub>  × *dp* × *C*<sub>*i*,m</sub>。如果这里有一部分哈希计算出了一个区块，那么其中会有一个竞争块成为孤块。被观察到的孤块哈希运算数量是 Space/*T*<sub>*i*</sub> × *C*<sub>*i*,o</sub>。如果我们忽略在同一个高度出现两个以上竞争块的小概率事件，可以得出：

![1559064685714](images/1559064685714.png)

namely

也就是

![1559064995366](images/1559064995366.png)



If we join this equation with Equation (2), we can solve for *dp*:

如果我们将这个公式和公式（2） 联立，可以得到 *dp*:

![1559065017925](images/1559065017925.png)

where *o<sub>i</sub>* is last epoch’s orphan rate.

在这里 *o<sub>i</sub>* 是上一个难度调节周期的孤块率。

#### 计算下一个难度调节周期的主链区块数

If the next epoch’s block propagation proceeds identically to the last epoch, the value *dp* should remain unchanged. In order to achieve the ideal orphan rate *o*<sub>ideal</sub> and the ideal epoch duration *L*<sub>ideal</sub>, following the same reasoning with Equation (4). We should have:

如果下一个难度调节周期的区块广播过程和上一个一致，*dp* 的值就会保持不变。为了能够得到理想的孤块率 *o*<sub>ideal</sub> 和理想的难度调节周期时长 *L*<sub>ideal</sub>，推导方式和等式（4）相同。我们应该有：

![1559065197341](images/1559065197341.png)



where ![1559065416713](images/1559065416713.png)is the number of main chain blocks in the next epoch, if our only goal is to achieve *o*<sub>ideal</sub> and *L*<sub>ideal</sub> . 

By joining Equation (4) and (5), we can solve for ![1559065488436](images/1559065416713.png):

在这里 ![1559065416713](images/1559065416713.png) 是下一个难度调节周期中主链区块数量，那么我们的唯一目标就是获取 *o*<sub>ideal</sub> 和 *L*<sub>ideal</sub> 值。 

通过将等式（4）和（5）联立起来，我们能够解出 ![1559065488436](images/1559065416713.png):

![1559065517956](images/1559065517956.png)



Now we can apply the upper and lower bounds to![1559065488436](images/1559065416713.png) and get *C*<sub>*i*+1,m</sub>:

现在我们可以设定 ![1559065488436](images/1559065416713.png) 的上下限，得到 *C*<sub>*i*+1,m</sub>:

![1559065670251](images/1559065670251.png)

Applying a lower bound ensures that an attacker cannot mine orphaned blocks deliberately to arbitrarily increase the block interval; applying an upper bound ensures that our protocol does not confirm more transactions than the capacity of most nodes.

设定下限能够保证攻击者无法通过故意挖出孤块来任意提高出块间隔；设定上限能够保障我们的协议不会确认超过大多数节点能力的交易数量。

#### 确定目标难度值

First, we introduce an adjusted orphan rate estimation ![1559065968791](images/1559065968791.png), which will be used to compute the target:

首先，我们引入一个经过调整的孤块率估值 ![1559065968791](images/1559065968791.png), 它会被用于计算这个难度调节目标：

![1559065997745](images/1559065997745.png)



Using ![1559065968791](images/1559065968791.png) instead of *o*<sub>ideal</sub> prevents some undesirable situations when the main chain block number reaches the upper or lower bound. Now we can compute *T*<sub>*i*+1</sub>:

使用 ![1559065968791](images/1559065968791.png) 代替 *o*<sub>ideal</sub> ，能够避免当主链区块数量达到上下限时，出到一些不可控的情况。现在我们能够计算出 *T*<sub>*i*+1</sub>:

![1559066101731](images/1559066101731.png)

where ![1559066131427](images/1559066131427.png) is the total hashes, ![1559066158164](images/1559066158164.png)is the total number of blocks. 

The denominator in Equation (7) is the number of hashes required to find a block.

其中 ![1559066131427](images/1559066131427.png) 是总哈希, ![1559066158164](images/1559066158164.png)是区块的总数. 

等式（7） 中的分母是找到一个区块所需要的哈希数量。

Note that if none of the edge cases are triggered, such as ![1559066233715](images/1559066233715.png)![1559066249700](images/1559066249700.png) or ![1559066329440](images/1559066329440.png)  , we can combine Equations (2), (6), and (7) and get:

注意，如果没有触发极端情况，比如 ![1559066233715](images/1559066233715.png)![1559066249700](images/1559066249700.png) 或 ![1559066329440](images/1559066329440.png)  , 我们能够联立等式（2），（6）和（7）并且得到：

![1559066373372](images/1559066373372.png)

This result is consistent with our intuition. On one hand, if the last epoch’s orphan rate *o*<sub>*i*</sub> is larger than the ideal value *o*<sub>ideal</sub>, the target lowers, thus increasing the difficulty of finding a block and raising the block interval if the total hash rate is unchanged. Therefore, the orphan rate is lowered as it is more unlikely to find a block during another block’s propagation. On the other hand, the target increases if the last epoch’s orphan rate is lower than the ideal value, decreasing the block interval and raising the system’s throughput.

结果和我们的直觉一致。一方面，如果上一个难度调节周期的孤块率 *o*<sub>*i*</sub> 大于理想值 *o*<sub>ideal</sub>，那么目标就会降低，这样在哈希率不变的情况下会提高找到一个区块的难度并且增加出块间隔。因此，由于在一个区块广播过程中找到另一个区块的概率降低了，那么孤块率就会降低。另一方面，如果上一个难度调节周期的孤块率比理想值低，那么目标难度就会增加，同时降低了出块间隔，并且提高系统的吞吐量。

#### 计算每个区块的奖励

Now we can compute the reward for each block:

现在我们可以计算每个区块的奖励:

![1559066526598](images/1559066526598.png)

The two cases differ only in the edge cases. The first case guarantees that the total reward issued in epoch *i* + 1 will not exceed R(*i* + 1).

这两种情况仅在边缘情况上有所不同。 第一种情况保证在难度调节周期 *i* + 1 中发行的总奖励不会超过 R(*i* + 1)。
