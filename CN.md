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
## 概述

Bitcoin's Nakamoto Consensus (NC) is well-received due to its simplicity and low communication overhead. However, NC suffers from two kinds of drawback: first, its transaction processing throughput is far from satisfactory; second, it is vulnerable to a selfish mining attack, where attackers can gain more block rewards by deviating from the protocol's prescribed behavior.

比特币的中本聪共识（NC）因其简单性和低通信开销而广受好评。然而，NC存在以下两种缺点：第一，其交易处理吞吐量远远不能令人满意;第二，它容易受到自私挖矿攻击，攻击者可以通过偏离协议规定的行为来获取更多的区块奖励。

The CKB consensus protocol is a variant of NC that raises its performance limit and selfish mining resistance while keeping its merits. By identifying and eliminating the bottleneck in NC's block propagation latency, our protocol supports very short block interval without sacrificing security. The shortened block interval not only raises the throughput, but also lowers the transaction confirmation latency. By incorporating all valid blocks in the difficulty adjustment, selfish mining is no longer profitable in our protocol.

CKB共识协议是NC共识协议的一种变体，它在保持其优点的同时提高了其性能极限和增大自私挖矿的阻力。通过识别并消除NC块传播延迟的瓶颈，我们的协议支持非常短的区块间隔，而不会牺牲安全性。缩短的区块间隔不仅提高了吞吐量，还降低了交易确认延迟。通过在难度调整中合并所有有效块，自私挖矿在我们的协议中不再有利可图。

<a name="Motivation"></a>
## 研究动机

Although a number of non-NC consensus mechanisms have been proposed, NC has the following threefold advantage comparing with its alternatives. First, its security is carefully scrutinized and well-understood [[1](https://www.cs.cornell.edu/~ie53/publications/btcProcFC.pdf), [2](https://eprint.iacr.org/2014/765.pdf), [3](https://fc16.ifca.ai/preproceedings/30_Sapirshtein.pdf), [4](https://eprint.iacr.org/2016/454.pdf), [5](https://eprint.iacr.org/2016/1048.pdf), [6](https://eprint.iacr.org/2018/800.pdf), [7](https://eprint.iacr.org/2018/129.pdf), [8](https://arxiv.org/abs/1607.02420)], whereas alternative protocols often open new attack vectors, either unintentionally [[1](http://fc19.ifca.ai/preproceedings/180-preproceedings.pdf), [2](https://www.esat.kuleuven.be/cosic/publications/article-3005.pdf)] or by relying on security assumptions that are difficult to realize in practice [[1](https://arxiv.org/abs/1711.03936), [2](https://arxiv.org/abs/1809.06528)]. Second, NC minimizes the consensus protocol's communication overhead. In the best-case scenario, propagating a 1MB block in Bitcoin is equivalent to broadcasting a compact block message of roughly 13KB [[1](https://github.com/bitcoin/bips/blob/master/bip-0152.mediawiki), [2](https://www.youtube.com/watch?v=EHIuuKCm53o)]; valid blocks are immediately accepted by all honest nodes. In contrast, alternative protocols often demand a non-negligible communication overhead to certify that certain nodes witness a block. For example, [Algorand](https://algorandcom.cdn.prismic.io/algorandcom%2Fa26acb80-b80c-46ff-a1ab-a8121f74f3a3_p51-gilad.pdf) demands that each block be accompanied by 300KB of block certificate. Third, NC's chain-based topology ensures that a transaction global order is determined at block generation, which is compatible with all smart contract programming models. Protocols adopting other topologies either [abandon the global order](https://allquantor.at/blockchainbib/pdf/sompolinsky2016spectre.pdf) or establish it after a long confirmation delay [[1](https://eprint.iacr.org/2018/104.pdf), [2](https://eprint.iacr.org/2017/300.pdf)], limiting their efficiency or functionality.

尽管已经提出了许多非NC共识机制，但NC与其替代方案相比具有以下三重优势。首先，它的安全性经过仔细审查和充分理解[[1](https://www.cs.cornell.edu/~ie53/publications/btcProcFC.pdf), [2](https://eprint.iacr.org/2014/765.pdf), [3](https://fc16.ifca.ai/preproceedings/30_Sapirshtein.pdf), [4](https://eprint.iacr.org/2016/454.pdf), [5](https://eprint.iacr.org/2016/1048.pdf), [6](https://eprint.iacr.org/2018/800.pdf), [7](https://eprint.iacr.org/2018/129.pdf), [8](https://arxiv.org/abs/1607.02420)],然而替代协议常常开放新的攻击矢量，无论是无意[[1](http://fc19.ifca.ai/preproceedings/180-preproceedings.pdf), [2](https://www.esat.kuleuven.be/cosic/publications/article-3005.pdf)] 还是依靠其安全性在实践中难以实现的假设[[1](https://arxiv.org/abs/1711.03936), [2](https://arxiv.org/abs/1809.06528)]。其次，NC最小化了共识协议的通信开销。在最理想的情况下，在比特币中传播1MB的块相当于广播大约13KB的压缩块[[1](https://github.com/bitcoin/bips/blob/master/bip-0152.mediawiki), [2](https://www.youtube.com/watch?v=EHIuuKCm53o)];所有诚实节点都会立即接受有效块。相反，替代协议通常需要不可忽略的通信开销来证明某些节点见证了块。例如，[Algorand](https://algorandcom.cdn.prismic.io/algorandcom%2Fa26acb80-b80c-46ff-a1ab-a8121f74f3a3_p51-gilad.pdf) 要求每个块都附带300KB的块证书。第三，NC的基于链的拓扑确保在块生成时确定事务全局顺序，这与所有智能合约编程模型兼容。采用其他拓扑结构的协议要么[放弃全局交易](https://allquantor.at/blockchainbib/pdf/sompolinsky2016spectre.pdf)，要么在经过长时间的确认延迟[[1](https://eprint.iacr.org/2018/104.pdf), [2](https://eprint.iacr.org/2017/300.pdf)]后才确认它，限制了其效率或功能。

Despite NC's merits, a scalability barrier hinders it from processing more than a few transactions per second. Two parameters collectively cap the system's throughput: the maximum block size and the expected block interval. For example, Bitcoin enforces a roughly 4MB block size upper bound and targets a 10-minute block interval and  with its **difficulty adjustment mechanism**, translating to roughly ten transactions per second (TPS). Increasing the block size or reducing the block interval leads to longer block propagation latency or more frequent block generation events, respectively; both approaches raise the fraction of blocks generated during other blocks' propagation, thus raising the fraction of competing blocks. As at most one block among the competing ones contributes to transaction confirmation, the nodes' bandwidth on propagating other **orphaned blocks** is wasted, limiting the system's effective throughput. Moreover, raising the orphan rate downgrades the protocol's security by lowering the difficulty of double-spending attacks [[1](<https://fc15.ifca.ai/preproceedings/paper_30.pdf>), [2](<https://fc15.ifca.ai/preproceedings/paper_101.pdf>)].

尽管NC有很多优点，但其可扩展性差阻碍了它每秒处理多笔交易。两个参数共同限制系统的吞吐量：最大区块容量和预期的块间隔。例如，比特币强制执行大约4MB的区块容量上限，并以10分钟的块间隔为目标，并且具有**动态难度调整机制**，换算为大约每秒10笔交易（TPS）。增加区块容量或减小区块间隔会导致更长时间的区块传播延迟或更频繁的出块;两种方法都提高了在其他块区传播期间生成的新块的比例，从而提高了竞争区块的比例。由于竞争对手中的最多只有一个块有助于交易确认，因此浪费了传播其他**孤块**的节点带宽，从而限制了系统的有效吞吐量。此外，提高孤块率会降低双花攻击的难度[[1](<https://fc15.ifca.ai/preproceedings/paper_30.pdf>), [2](<https://fc15.ifca.ai/preproceedings/paper_101.pdf>)]，从而降低协议的安全性。

Moreover, the security of NC is undermined by a [**selfish mining attack**](https://www.cs.cornell.edu/~ie53/publications/btcProcFC.pdf), which allows attackers to gain unfair block rewards by deliberately orphaning blocks mined by other miners. Researchers observe that the unfair profit roots in NC's difficulty adjustment mechanism, which neglects orphaned blocks when estimating the network's computing power. Through this mechanism, the increased orphan rate caused by selfish mining leads to lower mining difficulty, enabling the attacker's higher time-averaged block reward [[1](https://eprint.iacr.org/2016/555.pdf), [2](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-100.md), [3](https://arxiv.org/abs/1805.08281)].

此外，NC的安全性受到[**自私挖矿攻击**](https://www.cs.cornell.edu/~ie53/publications/btcProcFC.pdf)的破坏，这使得攻击者可以通过故意孤立的其他矿工开采的块从而获得不公平的区块奖励。研究人员观察到，不公平的利润源于NC的难度调整机制，该机制在估计网络的计算能力时忽略了孤块的障碍。通过这种机制，自私挖矿导致的孤儿率上升导致挖矿难度降低，使攻击者获得高于平均的区块奖励[[1](https://eprint.iacr.org/2016/555.pdf), [2](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-100.md), [3](https://arxiv.org/abs/1805.08281)]。

In this RFC, we present the CKB consensus protocol, a consensus protocol that raises NC's performance limit and selfish mining resistance while keeping all NC's merits. Our protocol supports very short block interval by reducing the block propagation latency. The shortened block interval not only raises the blockchain's throughput, but also minimizes the transaction confirmation latency without decreasing the level of confidence, as the orphan rate remains low. Selfish mining is no longer profitable as we incorporate all blocks, including uncles, in the difficulty adjustment when estimating the network's computing power, so that the new difficulty is independent of the orphan rate.

在此意见稿中，我们提出了CKB共识协议，该共识协议提高了NC的性能极限和增大自私挖矿的阻力，同时保留了所有NC的优点。我们的协议通过减少区块传播延迟来支持非常短的区块间隔。缩短的块间隔不仅提高了区块链的吞吐量，而且在不降低信任等级的情况下最小化了交易确认延迟，因为孤块率可以保持较低范围。自私挖矿不再有利可图，因为我们在估算网络的计算能力时将所有块（包括叔块）纳入难度调整中，使新难度与孤块率无关。

<a name="Technical-Overview"></a>
## 技术概述

Our consensus protocol makes three changes to NC.

我们的共识协议对NC进行了三个优化。

<a name="#Eliminating-the-Bottleneck-in-Block-Propagation"></a>
### 消除区块传播瓶颈

[Bitcoin's developers identify](https://www.youtube.com/watch?v=EHIuuKCm53o) that when the block interval decreases, the bottleneck in block propagation latency is transferring **fresh transactions**, which are newly broadcast transactions that have not finished propagating to the network when embedded in the latest block. Nodes that have not received these transactions must request them before forwarding the block to their neighbors. The resulted delay not only limits the blockchain's performance, but can also be exploited in a **de facto selfish mining attack**, where attackers deliberately embed fresh transactions in their blocks, hoping that the longer propagation latency gives them an advantage in finding the next block to gain more rewards.


[比特币的开发人员发现](https://www.youtube.com/watch?v=EHIuuKCm53o)，当区块间隔缩短时，区块传播延迟的瓶颈就是传递**新的交易**，这些新交易是在生成最新块时，尚未完成广播到网络的新广播交易。没有收到这些交易的节点必须在将块转发给其他节点之前请求它们。由此产生的延迟不仅限制了区块链的性能，而且还可以在**自私挖矿攻击**中被利用，攻击者故意在其块中嵌入新的交易，希望较长的传播延迟使他们有机会找到下一个块来获取更多奖励。

Departing from this observation, our protocol eliminates the bottleneck by decoupling NC's transaction confirmation into two separate steps: **propose** and **commit**. A transaction is proposed if its truncated hash, named `txpid`, is embedded in the **proposal zone** of a blockchain block or its **uncles**---orphaned blocks that are referred to by the blockchain block. Newly proposed transactions affect neither the block validity nor the block propagation, as a node can start transferring the block to its neighbors before receiving these transactions. The transaction is committed if it appears in the **commitment zone** in a window starting several blocks after its proposal. This two-step confirmation rule eliminates the block propagation bottleneck, as committed transactions in a new block are already received and verified by all nodes when they are proposed. The new rule also effectively mitigates de facto selfish mining by limiting the attack time window.

<a name="Utilizing-the-Shortened-Latency-for-Higher-Throughput"></a>
### 利用缩短的延迟提高吞吐量

Our protocol prescribes that blockchain blocks refer to all orphaned blocks as uncles. This information allows us to estimate the current block propagation latency and dynamically adjust the expected block interval, increasing the throughput when the latency improves. Accordingly, our difficulty adjustment targets a fixed orphan rate to utilize the shortened latency without compromising security. The protocol hard-codes the upper and lower bounds of the interval to defend against DoS attacks and avoid overloading the nodes. In addition, the block reward is adjusted proportionally to the expected block interval within an epoch, so that the expected time-averaged reward is independent of the block interval.

我们的协议规定区块将所有孤立的块称为叔块。此信息允许我们估计当前块传播延迟并动态调整预期的区块间隔，从而在延迟改善时增加吞吐量。因此，我们的难度调整目标是一个固定的孤块率，以利用缩短的延迟而不会影响安全性。该协议对间隔的上限和下限进行硬编码，以防止DoS攻击并避免节点过载。另外，块奖励与间隔内的预期区块间隔成比例地调整，使得预期时间平均奖励独立于区块间隔。

<a name="Mitigating-Selfish-Mining-Attacks"></a>
### 避免自私挖矿攻击

Our protocol incorporate all blocks, including uncles, in the difficulty adjustment when estimating the network's computing power, so that the new difficulty is independent of the orphan rate, following the suggestion of [Vitalik](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-100.md), [Grunspan and Perez-Marco](https://arxiv.org/abs/1805.08281).

根据[Vitalik](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-100.md), [Grunspan 和 Perez-Marco](https://arxiv.org/abs/1805.08281).的建议，我们的协议在估算网络的算力时，在难度调整中包含所有块，包括块，以便使新的难度与孤块率无关。

In addition, we prove that selfish mining is no longer profitable in our protocol. This prove is non-trivial as Vitalik, Grunspan and Perez-Marco's informal arguments do not rule out the possibility that the attacker adapts to the modified mechanism and still gets unfair block reward. For example, the attacker may temporarily turn off some mining gears in the first epoch, causing the modified difficulty adjustment algorithm to underestimate the network's computing power, and starts selfish mining in the second epoch for a higher overall time-averaged reward. We prove that in our protocol, selfish mining is not profitable regardless of how the attacker divides its mining power among honest mining, selfish mining and idle, and how many epochs the attack involves. The detailed proof will be released later.

此外，我们证明自私挖矿在我们的协议中不再有利可图。这个证明是意义重大的，因为Vitalik，Grunspan和Perez-Marco的非正式论证并不能排除攻击者适应修改后的机制并仍然获得不公平的区块奖励的可能性。例如，攻击者可能在第一个周期内暂时关闭一些采矿装备，导致修改后的难度调整算法低估网络的计算能力，并在第二个时期开始自私挖矿以获得更高的总体时间平均奖励。我们证明，在我们的协议中，无论攻击者如何将其挖矿能力划分为诚实的挖矿，自私挖矿和闲置，以及攻击涉及多少个周期，自私采矿都无利可图。详细证明将在稍后公布。

<a name="Specification"></a>
## 规范

<a name="Two-Step-Transaction-Confirmation"></a>
### 两步交易确认

In our protocol, we use a two-step transaction confirmation to eliminate the aforementioned block propagation bottleneck, regardless of how short the block interval is. We start by defining the two steps and the block structure, and then introduce the new block propagation protocol. 

在我们的协议中，我们使用两步交易确认来消除上述区块传播瓶颈，无论区块间隔有多短。我们首先定义两个步骤和区块结构，然后介绍新的区块传播协议。

#### 定义

> **Definition 1:** A transaction’s proposal id `txpid` is defined as the first *l* bits of the transaction hash `txid`.

In our protocol, `txpid` does not need to be as globally unique as `txid`, as a `txpid` is used to identify a transaction among several neighboring blocks. Since we embed `txpid`s in both blocks and compact blocks, sending only the truncated `txid`s could reduce the bandwidth consumption. 

When multiple transactions share the same `txpid`s, all of them are considered proposed. In practice, we can set *l* to be large enough so that the computational effort of finding a collision is non-trivial.

> **Definition 2:** A block *B*<sub>1</sub> is considered to be the *uncle* of another block *B*<sub>2</sub> if all of the following conditions are met:
>​	(1) *B*<sub>1</sub> and *B*<sub>2</sub> are in the same epoch, sharing the same difficulty;
>​	(2) height(*B*<sub>2</sub>) > height(*B*<sub>1</sub>);
>​	(3) *B*<sub>2</sub> is the first block in its chain to refer to *B*<sub>1</sub>. 

Our uncle definition is different from [that of Ethereum](https://github.com/ethereum/wiki/wiki/White-Paper#modified-ghost-implementation), in that we do not consider how far away the two blocks' first common ancestor is, as long as the two blocks are in the same epoch.

> **Definition 3:** A transaction is *proposed* at height *h*<sub>p</sub> if its `txpid` is in the proposal zone of the main chain block with height *h*<sub>p</sub> and this block’s uncles. 

It is possible that a proposed transaction is previously proposed, in conflict with other transactions, or even malformed. These incidents do not affect the block’s validity, as the proposal zone is used to facilitate transaction synchronization.

> **Definition 4:** A non-coinbase transaction is *committed* at height *h*<sub>c</sub> if all of the following conditions are met: 
> ​	(1) the transaction is proposed at height *h*<sub>p</sub> of the same chain, and *w<sub>close</sub>  ≤  h<sub>c</sub> − h*<sub>p</sub>  ≤  *w<sub>far</sub>*
> ​	(2) the transaction is in the commitment zone of the main chain block with height *h*<sub>c</sub>; 
> ​	(3) the transaction is not in conflict with any previously-committed transactions in the main chain. 
> The coinbase transaction is committed at height *h*<sub>c</sub> if it satisfies (2).

*w<sub>close</sub>* and *w<sub>far</sub>* define the closest and farthest on-chain distance between a transaction’s proposal and commitment. We require *w<sub>close</sub>*  to be large enough so that *w<sub>close</sub>* block intervals are long enough for a transaction to be propagated to the network. 

These two parameters are also set according to the maximum number of transactions in the proposed transaction pool of a node’s memory. As the total number of proposed transactions is limited, they can be stored in the memory so that there is no need to fetch a newly committed transaction from the hard disk in most occasions. 

A transaction is considered embedded in the blockchain when it is committed. Therefore, a receiver that requires σ confirmations needs to wait for at least *w<sub>close</sub>* +σ blocks after the transaction is broadcast to have confidence in the transaction. 

In practice, this *w<sub>close</sub>* - block extra delay is compensated by our protocol’s shortened block interval, so that the usability is not affected.

#### Block and Compact Block Structure

A block in our protocol includes the following fields:

| Name            | Description                          |
| :-------------- | :----------------------------------- |
| header          | block metadata                       |
| commitment zone | transactions committed in this block |
| proposal zone   | `txpid`s proposed in this block      |
| uncle headers   | headers of uncle blocks              |
| uncles’ proposal zones   | `txpid`s proposed in the uncles              |

Similar to NC, in our protocol, a compact block replaces a block’s commitment zone with the transactions’ `shortid`s, a salt and a list of prefilled transactions. All other fields remain unchanged in [the compact block](https://github.com/bitcoin/bips/blob/master/bip-0152.mediawiki).

Additional block structure rules:

- The total size of the first four fields should be no larger than the hard-coded **block size limit**. The main purpose of implementing a block size limit is to avoid overloading public nodes' bandwidth. The uncle blocks’ proposal zones do not count in the limit as they are usually already synchronized when the block is mined. 
- The number of `txpid`s in a proposal zone also has a hard-coded upper bound.

Two heuristic requirements may help practitioners choose the parameters. First, the upper bound number of `txpid`s in a proposal zone should be no smaller than the maximum number of committed transactions in a block, so that even if *w<sub>close</sub>=w<sub>far</sub>*, this bound is not the protocol's throughput bottleneck. Second, ideally the compact block should be no bigger than 80KB. According to [a 2016 study by Croman et al.](https://fc16.ifca.ai/bitcoin/papers/CDE+16.pdf), messages no larger than 80KB have similar propagation latency in the Bitcoin network; larger messages propagate slower as the network throughput becomes the bottleneck. This number may change as the network condition improves.

#### Block Propagation Protocol

In line with [[1](https://www.cs.cornell.edu/~ie53/publications/btcProcFC.pdf), [2](https://arxiv.org/abs/1312.7013), [3](https://eprint.iacr.org/2014/007.pdf)], nodes should broadcast all blocks with valid proofs-of-work, including orphans, as they may be referred to in the main chain as uncles. Valid proofs-of-work cannot be utilized to pollute the network, as constructing them is time-consuming. 

Our protocol’s block propagation protocol removes the extra round trip of fresh transactions in most occasions. When the round trip is inevitable, our protocol ensures that it only lasts for one hop in the propagation. This is achieved by the following three rules: 

1. If some committed transactions are previously unknown to the sending node, they will be embedded in the prefilled transaction list and sent along with the compact block. This only happens in a de facto selfish mining attack, as otherwise transactions are synchronized when they are proposed. This modification removes the extra round trip if the sender and the receiver share the same list of proposed, but-not-broadcast transactions. 
2. If certain committed transactions are still missing, the receiver queries the sender with a short timeout. Triggering this mechanism requires not only a successful de facto selfish mining attack, but also an attack on transaction propagation to cause inconsistent proposed transaction pools among the nodes. Failing to send these transactions in time leads to the receiver disconnecting and blacklisting the sender. Blocks with incomplete commitment zones will not be propagated further.

3. As long as the commitment zone is complete and valid, a node can start forwarding the compact block before receiving all newly-proposed transactions. In our protocol, a node requests the newly-proposed transactions from the upstream peer and sends compact blocks to other peers simultaneously. This modification does not downgrade the security as transactions in the proposal zone do not affect the block’s validity.


The first two rules ensure that the extra round trip caused by a de facto selfish mining attack never lasts for more than one hop.

<a name="Dynamic-Difficulty-Adjustment-Mechanism"></a>
### 动态难度调整机制

我们修改了Nakamoto 共识难度调整机制，以便: (1) 自私挖矿不再有利可图; (2) 根据网络的带宽和延迟动态调整吞吐量。实现目标1, 我们的协议在计算上一个时期的**调整后的哈希率估计**时包含所有块而不是仅主链, ，其确定每个奖励单元的下一个时期所需的计算工作量. 实现目标2, 我们的协议计算下一个时期中具有最后一个时期的孤儿率的主链块的数量。然后通过组合这些结果来计算块奖励和目标。

引入了附加约束以最大化协议的兼容性:

1. 所有时期具有相同的预期长度 *L<sub>ideal</sub>*, 并且在时期 R(*i*) 中发布的最大块奖励仅取决于时期数 *i*, 因此动态块间隔不会使奖励发布策略复杂化. 

2. 几个上限和下限应用于哈希率估算和主链块的数量，因此我们的协议不会损害网络的去中心化或抗攻击性。

#### Notations

Similar to Nakamoto Consensus , our protocol’s difficulty adjustment algorithm is executed at the end of every epoch. It takes four inputs:

| Name            | Description                          |
| :-------------- | :----------------------------------- |
| *T*<sub>*i*</sub>          | Last epoch’s target                       |
| *L*<sub>*i*</sub> | Last epoch’s duration: the timestamp difference between epoch *i* and epoch (*i* − 1)’s last blocks |
| *C*<sub>*i*,m</sub>   | Last epoch’s main chain block count      |
| *C*<sub>*i*,o</sub>   | Last epoch’s orphan block count:  the number of uncles embedded in epoch *i*’s main chain         |

Among these inputs, *T<sub>i</sub>* and *C*<sub>*i*,m</sub> are determined by the last iteration of difficulty adjustment; *L*<sub>*i*</sub> and *C*<sub>*i*,o</sub> are measured after the epoch ends. The orphan rate *o*<sub>*i*</sub> is calculated as *C*<sub>*i*,o</sub> / *C*<sub>*i*,m</sub>. We do not include *C*<sub>*i*,o</sub> in the denominator to simplify the equation. As some orphans at the end of the epoch might be excluded from the main chain by an attack, *o*<sub>*i*</sub> is a lower bound of the actual number. However, [the proportion of deliberately excluded orphans is negligible](https://eprint.iacr.org/2014/765.pdf) as long as the epoch is long enough, as the difficulty of orphaning a chain grows exponentially with the chain length. 

The algorithm outputs three values:

| Name            | Description                          |
| :-------------- | :----------------------------------- |
| *T*<sub>*i*+1</sub>          | Next epoch’s target                       |
| *C*<sub>i+1,m</sub> | Next epoch’s main chain block count |
| *r*<sub>*i*+1</sub>   | Next epoch’s block reward     |

If the network hash rate and block propagation latency remains constant, *o*<sub>*i*+1</sub> should reach the ideal value *o*<sub>ideal</sub>, unless *C*<sub>*i*+1,m</sub> is equal to its upper bound *C*<sub>m</sub><sup>max</sup>  or its lower bound *C*<sub>m</sub><sup>min</sup> . Epoch *i* + 1 ends when it reaches *C*<sub>*i*+1,m</sub> main chain blocks, regardless of how many uncles are embedded.

#### Computing the Adjusted Hash Rate Estimation

The adjusted hash rate estimation, denoted as *HPS<sub>i</sub>* is computed by applying a dampening factor τ to the last epoch’s actual hash rate ![1559068235154](images/1559068235154.png). The actual hash rate is calculated as follows:

![1559064934639](images/1559064934639.png)

where:

- HSpace is the size of the entire hash space, e.g., 2^256 in Bitcoin,
- HSpace/*T<sub>i</sub>* is the expected number of hash operations to find a valid block, and 
- *C*<sub>*i*,m</sub> + *C*<sub>*i*,o</sub> is the total number of blocks in epoch *i*

![1559068266162](images/1559068266162.png) is computed by dividing the expected total hash operations with the duration *L<sub>i</sub>*

Now we apply the dampening filter:

![1559064108898](images/1559064108898.png)

where *HPS*<sub>*i*−1</sub> denotes the adjusted hash rate estimation output by the last iteration of the difficulty adjustment algorithm. The dampening factor ensures that the adjusted hash rate estimation does not change more than a factor of τ between two consecutive epochs. This adjustment is equivalent to the Nakamoto Consensus application of a dampening filter. Bounding the adjustment speed prevents the attacker from arbitrarily biasing the difficulty and forging a blockchain, even if some victims’ network is temporarily controlled by the attacker.

#### Modeling Block Propagation

It is difficult, if not impossible, to model the detailed block propagation procedure, given that the network topology changes constantly over time. Luckily, for our purpose, it is adequate to express the influence of block propagation with two parameters, which will be used to compute *C*<sub>*i*+1,m</sub>  later.

We assume all blocks follow a similar propagation model, in line with [[1](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.395.8058&rep=rep1&type=pdf), [2](https://fc16.ifca.ai/bitcoin/papers/CDE+16.pdf)]. In the last epoch, it takes *d* seconds for a block to be propagated to the entire network, and during this process, the average fraction of mining power working on the block’s parent is *p*. Therefore, during this *d* seconds, *HPS*<sub>*i* </sub> × *dp* hash operations work on the parent, thus not contributing to extending the blockchain, while the rest *HPS*<sub>*i*</sub> × *d*(1 − *p*) hashes work on the new block. Consequently, in the last epoch, the total number of hashes that do not extend the blockchain is *HPS*<sub>*i*</sub>  × *dp* × *C*<sub>*i*,m</sub>. If some of these hashes lead to a block, one of the competing blocks will be orphaned. The number of hash operations working on observed orphaned blocks is HSpace/*T*<sub>*i*</sub> × *C*<sub>*i*,o</sub>. If we ignore the rare event that more than two competing blocks are found at the same height, we have:

![1559064685714](images/1559064685714.png)

namely

![1559064995366](images/1559064995366.png)



If we join this equation with Equation (2), we can solve for *dp*:

![1559065017925](images/1559065017925.png)

where *o<sub>i</sub>* is last epoch’s orphan rate.

#### Computing the Next Epoch’s Main Chain Block Number
If the next epoch’s block propagation proceeds identically to the last epoch, the value *dp* should remain unchanged. In order to achieve the ideal orphan rate *o*<sub>ideal</sub> and the ideal epoch duration *L*<sub>ideal</sub>, following the same reasoning with Equation (4). We should have:

![1559065197341](images/1559065197341.png)



where ![1559065416713](images/1559065416713.png)is the number of main chain blocks in the next epoch, if our only goal is to achieve *o*<sub>ideal</sub> and *L*<sub>ideal</sub> . 

By joining Equation (4) and (5), we can solve for ![1559065488436](images/1559065416713.png):

![1559065517956](images/1559065517956.png)



Now we can apply the upper and lower bounds to![1559065488436](images/1559065416713.png) and get *C*<sub>*i*+1,m</sub>:

![1559065670251](images/1559065670251.png)

Applying a lower bound ensures that an attacker cannot mine orphaned blocks deliberately to arbitrarily increase the block interval; applying an upper bound ensures that our protocol does not confirm more transactions than the capacity of most nodes.

#### 测定目标难度

首先，我们介绍一个调整后的孤儿率估算 ![1559065968791](images/1559065968791.png), 这将用于计算目标:

![1559065997745](images/1559065997745.png)



使用 ![1559065968791](images/1559065968791.png) 代替 *o*<sub>ideal</sub> 防止当当主链块编号达到上限或下限时出现一些不良情况。 现在我们可以计算 *T*<sub>*i*+1</sub>:

![1559066101731](images/1559066101731.png)

这里的 ![1559066131427](images/1559066131427.png) 是总哈希, ![1559066158164](images/1559066158164.png)是区块的总数. 

方程 (7) 中的分母 是找到区块所需的哈希数量.

注意，如果没有触发任何边缘情况, 如 ![1559066233715](images/1559066233715.png)![1559066249700](images/1559066249700.png) 或 ![1559066329440](images/1559066329440.png)  , 我们可以结合方程 (2), (6), 和 (7) 并得到：

![1559066373372](images/1559066373372.png)



这个结果与我们的直觉一致。 一方面，如果最后一个时期的孤儿率 *o*<sub>*i*</sub> 大于理想值 *o*<sub>ideal</sub>, 则目标降低，因此如果总哈希率不变则增加找到块的难度并增加块间隔. 则目标降低，因此如果总哈希率不变则增加找到块的难度并增加块间隔。 另一方面，如果最后一个时期的孤儿率低于理想值，则目标增加，减少了块间隔并提高了系统的吞吐量。

#### Computing the Reward for Each Block

Now we can compute the reward for each block:

![1559066526598](images/1559066526598.png)

The two cases differ only in the edge cases. The first case guarantees that the total reward issued in epoch *i* + 1 will not exceed R(*i* + 1).
