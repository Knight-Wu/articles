# bitcoin whitepaper
## Transaction or blockchain
一位所有者将一枚硬币交给另一个人的时候，要通过在这个数字签名链的末尾附加上以下数字签名：上一笔交易的哈希（hash，音译，亦翻译为“散列值”），以及新所有者的公钥。收款人可以通过验证签名去验证数字签名链的所属权。

## timestamp server
时间戳服务器是这样工作的：为一组（block）记录（items）的哈希打上时间戳，而后把哈希广播出去, 时间戳能够证明那数据在那个时间点之前已然存在，否则那哈希也就无法生成。每个时间戳在其哈希中包含着之前的时间戳，因此构成了一个链；每一个新的时间戳被添加到之前的时间戳之后。
## POW
所谓的工作证明, 不断在区块之中增加一个随机数（Nonce），直到这个区块的哈希以指定数量的 0 开头。每增加一个 0 的要求，将使得工作量指数级增加，并且，这个工作量的验证却只需通过计算一个哈希。一旦 CPU 的耗费算力所获的的结果满足工作证明，那么这个区块将不再能被更改，除非重新完成之前的所有工作量。随着新的区块不断被添加进来，改变当前区块即意味着说要重新完成所有其后区块的工作。
</br>
为了应对硬件算力综合的不断增加，以及随着时间推进可能产生的节点参与数量变化，工作证明难度由此决定：基于平均每小时产生的区块数量的一个移动平均值。如果区块生成得过快，那么难度将会增加

## 如何添加交易进入新区块
1. 所有新的交易向所有节点广播；
2. 每个节点将新交易打包到一个区块；
3. 每个节点开始为此区块找工作证明；
4. 当某个区块找到其工作证明，它就要将此区块广播给相邻节点, 再由p2p 网络扩散到全网节点. 
5. 众多其他节点当且只当以下条件满足才会接受这个区块：其中所有的交易都是有效的，检查UTXO 是否合法, 未被双重支付等；
6. 众多节点向网络表示自己接受这个区块的方法是，在创建下一个区块的时候，把被接受区块的哈希当作新区块之前的哈希。

</br>
新的交易不见得一定要广播到达所有的节点。只要到达足够多的节点，那么没多久这些交易就会被打包进一个区块。区块广播也容许一些消息被丢弃。如果一个节点并未接收到某个区块，那么这个节点会在它接收到下一个区块的时候意识到自己错失了之前的区块，因此会发出补充那个遗失区块的请求。

* 如何解决分叉

可能在某些场景会出现有分叉的情况, 如何在分叉间选择的细节问题可能要看代码, 但最后会由最长链胜出


## 矿工的奖励
按照约定，每个区块的第一笔交易是一个特殊的交易，它会生成一枚新的硬币，所属权是这个区块的生成者(矿工)。这么做，让节点有所奖励，也提供了一种将硬币发行到流通之中的方式

### 奖励机制保证诚实
奖励机制也可能会鼓励节点保持诚实。如果一个贪婪的攻击者能够网罗比所有诚实节点都更多的 CPU 算力，他必须做出一个选择：是用这些算力通过把自己花出去的钱偷回来去欺骗别人呢？还是用这些算力去生成新的硬币？他应该能够发现按照规则行事是更划算的，当前规则使得他能够获得比所有其他人加起来都更多的硬币，(因为算力最多)

## bitcoin SPV
1. 下载区块头
区块头结构：每个区块头包含以下信息（共80字节）：

版本号

前一个区块的哈希值（链式结构）

默克尔根（Merkle Root）

时间戳

难度目标

Nonce（用于工作量证明）

存储需求：截至2023年，约40万区块头仅需约40MB存储空间。

2. 验证区块的工作量证明（PoW）是否有效
SPV节点检查每个区块头的哈希值是否符合当前的难度目标(从前面数有多少个0)，确保区块是由矿工通过算力竞争生成的合法区块。

最长链规则：节点选择累积工作量最大的链作为有效链。

3. 获取交易的默克尔路径
当用户需要验证某笔交易时，向全节点请求以下信息：

交易所在的区块：确认该区块已被网络接受。

默克尔路径：从目标交易到默克尔根的哈希路径。

验证过程：

使用默克尔路径中的哈希值逐层计算，最终得到默克尔根。

将计算结果与区块头中的默克尔根比对，若一致则证明交易存在于该区块中。

4. 确认交易深度
确认数（Confirmations）：交易所在区块之后新增的区块数量。通常6个确认（约1小时）后视为不可逆转。

# MEV 攻击(Maximal Extractable Value)
validator 或者矿工操纵区块链的交易顺序或插入特定的交易, 获取价值的行为; 通常需要交易的排序权限, 
## 三明治攻击
攻击者检测到用户提交了一笔大额DEX交易（如Uniswap上的ETH买入）。

攻击者在该交易前插入一笔高价买入交易，推高市场价格。

用户交易以更高价格成交。

攻击者随后卖出ETH，赚取差价。

影响：用户交易滑点增加，实际成交价格劣化。
# 51 % 攻击
The attacker's strategy is simple:

1. Send 100 BTC to a merchant in exchange for some product (preferably a rapid-delivery digital good)
2. Wait for the delivery of the product
3. Produce another transaction sending the same 100 BTC to himself
4. Try to convince the network that his transaction to himself was the one that came first.
   
Once step (1) has taken place, after a few minutes some miner will include the transaction in a block, say block number 270000. After about one hour, five more blocks will have been added to the chain after that block, with each of those blocks indirectly pointing to the transaction and thus "confirming" it. At this point, the merchant will accept the payment as finalized and deliver the product; since we are assuming this is a digital good, delivery is instant. Now, the attacker creates another transaction sending the 100 BTC to himself. If the attacker simply releases it into the wild, the transaction will not be processed; miners will attempt to run APPLY(S,TX) and notice that TX consumes a UTXO which is no longer in the state. So instead, the attacker creates a "fork" of the blockchain, starting by mining another version of block 270000 pointing to the same block 269999 as a parent but with the new transaction in place of the old one. Because the block data is different, this requires redoing the proof-of-work. Furthermore, the attacker's new version of block 270000 has a different hash, so the original blocks 270001 to 270005 do not "point" to it; thus, the original chain and the attacker's new chain are completely separate. The rule is that in a fork the longest blockchain is taken to be the truth, and so legitimate miners will work on the 270005 chain while the attacker alone is working on the 270000 chain. In order for the attacker to make his blockchain the longest, he would need to have more computational power than the rest of the network combined in order to catch up (hence, "51% attack").

简单解释就是:
1. 先做了一笔交易, 假设花费100 BTC, 然后等收货之后需要再次使用这个100 BTC, 并把之前第一笔交易抹掉或者第二笔交易发生在前面
2. 假设第一笔交易已经在block 100 上了, 后面又有101, 102 block, 那么攻击者需要重新生成block 100, ... 102 , 那么他需要重新计算POW, 而且比其他任何分叉要计算的快, 否则他不能成为最长的那个
3. 那么为了成为最长的, 就需要掌握最多的算力, 只有他掌握51% 以上的算力时才能保证他的分支是最长的.

# No coin in genesis state
In order to compensate miners for this computational work, the miner of every block is entitled to include a transaction giving themselves 25 BTC out of nowhere. Additionally, if any transaction has a higher total denomination in its inputs than in its outputs, the difference also goes to the miner as a "transaction fee".

 Incidentally, this is also the only mechanism by which BTC are issued; the genesis state contained no coins at all.
BTC 的产生是在挖矿得到新区块的时候作为奖励给矿工, 创世的时候是没有比特币的. 
# UTXO(Unspent Transaction Output)
因为在比特币等系统中不存在账户-余额这种类似数据库记录的东西, 那么如何判断用户的余额是否够支出呢? 
每次交易前会根据用户地址查询关联的所有交易, 用总收入减总支出就是余额, 过程中可能需要合并多项UTXO 

UTXO：比特币网络中的每一笔交易（除创世区块外）都由**输入（Inputs）和输出（Outputs）**组成。输入是引用之前交易的输出（即UTXO），输出则是新生成的UTXO。

所有权验证：每个UTXO被锁定（加密脚本）到某个地址（公钥哈希）。只有拥有对应私钥的用户才能解锁UTXO，将其用作新交易的输入。

余额计算：用户的“余额”是所有归属于其地址的UTXO的总和。

* 如何快速查询 UTXO

比特币全节点在本地维护了一个UTXO数据库（称为UTXO集），记录了当前所有未花费的输出。当需要查询某个地址的UTXO时，全节点可以直接扫描UTXO集，而非整个区块链。具体流程如下：

UTXO集的结构：

每个UTXO条目包含：交易ID（TXID）、输出索引（Vout）、金额、锁定脚本（ScriptPubKey）。

查询流程：

将目标地址转换为对应的锁定脚本（如通过公钥哈希生成OP_DUP OP_HASH160 <公钥哈希> OP_EQUALVERIFY OP_CHECKSIG）。

在UTXO集中快速匹配所有锁定脚本符合该地址的条目。

汇总这些条目，得到该地址的UTXO列表。

UTXO集是高度优化的内存数据库，查询速度极快

* 如何更新UTXO , 例如用户的钱包

当新交易确认时，钱包更新本地UTXO集

# 为什么需要GAS
* 资源成本补偿

矿工执行合约代码需要消耗计算资源（CPU）、存储（内存/磁盘）和带宽。Gas 费用补偿这些成本。
每个 EVM 操作码（如 ADD, SSTORE）有预设的 gas 成本，反映其资源消耗（例如：SSTORE 写入存储的成本远高于 ADD）。

* 防止滥用与攻击
  
Gas 限制（Gas Limit）阻止无限循环或复杂计算阻塞网络（例如：若代码进入死循环，gas 耗尽后执行自动终止）。

* 市场调节机制
  
Gas 价格（Gas Price）由用户设定，矿工优先打包 gas 价格高的交易，形成市场驱动的优先级排序。
# 为什么要通过竞争来出块？
* 去中心化：让全球矿工公平竞争，而不是由一个中心化的机构控制交易。
* 安全性：计算哈希值的过程需要消耗大量算力，使得攻击网络变得非常昂贵。
* 防止篡改：如果想篡改过去的交易，需要重新计算所有后续区块的哈希值，几乎不可能。

# solana vs bitcoin vs eth

## POH vs POW vs POS

### POW
bitcoin POW 通过计算数学题来竞争出块，类似一个概率题，谁的算力越高，那么计算成功的概率越大，但是计算的过程很耗资源（电力），也很慢，
，区块时间：约 10 分钟。TPS（交易处理能力）：约 7 TPS。

### POS
每个validator 都必须质押32 eth， 出块的时候从所有的validator 中随机选出一个作为提出者（proposer），并选取128 个validator 作为委员会， 广播到所有validator， proposer 把交易打包成区块，广播给委员会，委员会中多数把区块验证通过后就加入到区块链中。
</br>
出块时间：平均 15 秒，TPS 目前约 15-30 TPS（和 PoW 时代差不多）。

#### 为什么POS 不需要算力证明还是这么慢

1. 需要得到委员会多数节点投票，投票和网络通信延迟
2. 交易只能串行执行， 因为evm 有一个全局状态结构体， 必须是单线程执行的
3. 每个区块大小有限，每个区块的gas 大概是3 千万，如果单个区块过大，导致节点间同步区块变慢，低带宽或硬件较差的节点会失联，可能会导致非中心化
4. 没有sharding，单个节点必须存储和验证所有交易

* 如何解决eth 过慢

1. 在L2 处理具体交易细节，把多笔交易的总结后的数据存eth， TPS 提升， 存储量降低， 例如 Arbitrum / Optimism
2. Sharding， 每个节点不存储全量数据。

### POH


#### POH 验证的过程是怎样的

假设：
	•	Leader  A  生成了一条哈希链  H_0 → H_1 → H_2 → H_3 。
	•	对应的交易是  E_1, E_2, E_3 。
</br>

广播的内容包含：
	1.	初始状态： H_0 （PoH 的起点）。
	2.	交易数据： E_1, E_2, E_3 。
	3.	哈希链： H_1, H_2, H_3 （每个  H_n  包含一个时间戳）
</br>
其他节点  B  和  C  的验证过程：
	1.	接收到  H_0, E_1, E_2, E_3  和  H_1, H_2, H_3 。
	2.	重新计算哈希链：
	•	 H_1 = SHA256(H_0 + E_1) 
	•	 H_2 = SHA256(H_1 + E_2) 
	•	 H_3 = SHA256(H_2 + E_3) 
	3.	比较计算结果和广播的哈希值是否一致。
	•	如果一致，说明事件顺序和时间戳合法。

#### POS 和 POH 的通信次数对比
 PoS：通信次数分析

在 PoS 中，节点必须通过 拜占庭容错协议（如 PBFT 或 Tendermint） 达成共识。主要步骤和通信如下：
	1.	提议阶段（Propose Phase）：
	•	出块者（Leader，例如  A ）提议一个区块并将其广播给所有节点  B, C, D 。
	•	通信：1 次广播，涉及  A → B, C, D 。

	2.	验证阶段（Pre-vote Phase）：
	•	所有节点独立验证提议区块的有效性（如交易是否合法，签名是否正确）。
	•	每个节点将其验证结果广播给其他所有节点。
	•	通信：每个节点广播给其他 3 个节点，共  4 * 3 = 12  次。

	3.	投票阶段（Pre-commit Phase）：
	•	节点收到投票后，再次投票是否接受区块。
	•	每个节点将其投票结果广播给其他节点。
	•	通信：每个节点再次广播给其他 3 个节点，共  4 * 3 = 12  次。

	4.	确认阶段（Commit Phase）：
	•	如果 2/3 的节点同意（在这里是至少 3 个节点同意），区块被确认。
	•	通信：确认消息广播给所有节点，涉及  4 * 3 = 12  次。

</br>
总通信次数：37

</br>
PoH：通信次数分析

在 PoH 中，Leader 独立生成时间顺序并广播哈希链。节点只需验证哈希链和交易数据，无需相互通信。主要步骤和通信如下：

	1.	Leader 生成哈希链并广播：
	•	Leader  A  生成哈希链  H_1, H_2, H_3  和交易数据，并将其广播给所有节点  B, C, D 。
	•	通信：1 次广播（Leader → 其他 3 个节点）。

	2.	节点独立验证：
	•	每个节点独立验证哈希链的正确性和交易的合法性，无需与其他节点通信。

	3.	区块确认：
	•	节点收到广播后，验证通过即确认区块，无需进一步通信。


# bitcoin 如何防止double spend
<img width="823" alt="image" src="https://github.com/user-attachments/assets/ba46bec6-7f69-4f44-b2fc-ab6408a09d3c" />

1. 攻击者A 首先花费了 10 btc， 交易a 记录在id=200 区块上， 然后已经被多数miner 所确认
2. 此时区块增长到了205， 攻击者此时维护了当前区块链副本，把id=200 的区块的交易a 修改成区块b，尝试修改之前已经发生的交易
3. 由于每个区块都包含之前区块计算结果的hash，所以攻击者需要计算 id=200 之后的所有区块，
4. 只有最长的区块链被认为是权威的，所以攻击者的算力必须超过其他所有节点才能保证新的区块都加到他这个副本，保证他的链是最长的，从而达到修改历史交易的目的

# Merkle tree

 ## Merkle Tree 解决了哪些问题？

![image](https://github.com/user-attachments/assets/5884896a-72c6-4056-85c4-45d6c53f6047)

1. 验证某个交易是否存在

如果验证一个交易是否存在于区块中，每次都需要扫描整个区块，效率低下。
Merkle Tree 提供了一种路径验证机制，仅通过一条路径（从叶节点到根节点）即可验证，复杂度从 O(N) 降到 O(log N)。
例如需要提供, 交易哈希
默克尔路径：从TXID到根哈希所需的所有中间哈希值。

* 验证步骤

轻节点使用TXID和默克尔路径逐层计算哈希，最终结果与区块头中的Merkle Root对比。

2. 如何在分布式节点中快速验证所有节点的数据是否一致

每个节点只需要比较 Merkle Root，而无需比较整个数据集合。

3. 仅验证的节点不需要下载全量数据（SPV（Simplified Payment Verification）），

只需要下载：Merkle Root 以及验证路径上的哈希值（Merkle Proof）。

# AMM
通过两个币的数量的比例关系，实现去中心化免中介的交易。

## uniswap v3
https://github.com/Dapp-Learning-DAO/Dapp-Learning/blob/main/defi/Uniswap-V3/whitepaperGuide/understandV3Witepaper.md

用户可以在提供LP 时指定价格范围, 提高资金使用率, 为什么能提升呢, 数学上可以证明, 但由于指定了价格范围, 当价格越过指定的范围时, 会完全转变成另一种资金, 无偿损失相比于不指定范围的会更高. 

### 如何计算手续费
* 传统思路
1. 当前价格出现在头寸的范围内时, 算出一笔交易的手续费, 并按照多个头寸的比例分.
2. 需要把每个头寸的当前手续费先从链上读出来, 再加上当前的变动,
3. 由于头寸是用户设置的, 分布再价格轴上的无限区间, gas 费会非常高

* 改进: 如果不指定价格范围, 假设头寸分布在所有价格区间

1. 有一个全局变量记录着全局手续费, 每次交易都累加这个值.
2. LP 的手续费只有他做mint 和 burn 时才计算, 按照他做LP 时手续费增加的量, 再乘以他所占的比例, 但是他所占的比例是一直变化的该怎么计算?

#### 最终改进
* tick pacing
把价格轴按等比数列拆分, 否则价格区间的保存将有无限多个范围, 如果波动率大的交易对, 就用价格轴拆分较为稀疏的方式, 相隔序号较多作为一个有效价格点, 如果波动率低就用相隔序号较低作为有效价格点, 价格的表示就更加精细. 
![image](https://github.com/user-attachments/assets/7576db11-891d-4b0f-82e0-c6b19691c391)

* 多个position 如何计算手续费呢(TODO)

