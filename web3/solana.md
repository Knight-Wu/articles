# Arch
![image](https://github.com/user-attachments/assets/d41e7d79-e33a-41dd-a6b5-3985e6b262c9)

![image](https://github.com/user-attachments/assets/9edafe9d-6a1d-42be-84ac-9a50b04564d8)

# solana 为什么没有类似eth 的mempool
1. 追求低延迟, 交易直接通过P2P 网络发送给leader, 避免全局mempool 带来的广播延迟. 

# solana 如何判断某个交易是否过期，过期时间是多少
https://solana.com/zh/docs/advanced/confirmation

1. 提交交易的时候需要带上latestBlockHash
2. validator 记录着最近151 个blockHash，如果latestBlockHash 在这151个hash 之中，则没有过期，否则过期
3. 一个block 的产生时间在400- 600ms 之间，所以transaction 的过期时间在60 to 90 seconds

# 为什么交易需要有过期时间
因为solana 靠这个来防止交易被重复处理

1. solana validator 在内存中记录着最近的交易列表，如果新来的交易与列表中的交易id 重复，则拒绝
2. 这个最近的交易列表由最近的151 个block 的交易构成，如果当前交易和151 之前的交易重复，会因为latestBlockHash 不在这151 block hash 中，被超时拒绝；
如果latestBlockHash 没超时，则与最近的交易列表中的交易判断，重复则拒绝。

# solana block commitment 区别
* finalized

指的是区块被大多数节点投票通过，并且保持了最大锁定时间，该区块被认为不可逆

* confirmed

同样是区块被大多数节点投票通过，不算该区块的后续子区块的投票

# Bond（押金），slashing（惩罚），super majority
每个validator 必须抵押一定量的币才能参与到网络投票中，可以类比成POW 中的电力和硬件；如果支持错误的分叉链，押金会被部分没收；
当投票超过网络中总节点的三分之二时（押金作为权重计算）网络达成共识，所以只有当攻击者想要攻击必须要付出三分之一的网络代价。

# Leader 选举
## 如何选出leader
1. 采用POS 和轮转机制来选出leader，所有质押sol 的validator 都有可能成为leader，质押越多的概率越高。
2. 每几个slot（时间单位，400MS，bitcoin 是十分钟，eth 是12秒）会有一个leader，每个epoch 周期由多个slot 组成，每次开始新的epoch 会事先计算好每个slot 对应的leader，每个slot 会生成一个block, block time 是400 MS

## 为什么需要有epoch 
一个epoch 大约为2.5 天，每次质押的sol 变化将在下个epoch 生效，每次epoch 结束时会清理不活跃的账户并收取租金。

## 选出leader 之后
1. 接受交易请求，并将交易打包成区块，计算新的POH 链
2. 将新的区块广播给其他validator 验证, 由于POH 链的验证可以并行, 所以相比单线程生成POH 会快很多.
3. validator 通过tower BFT 进行共识，多数质押权重的validator 认可生成的新的POH 链，则该区块被确认；
4. 如果链产生了多个分叉，哪个分叉得到的投票权重高则保留。

## 交易请求如何发送到leader
1. 交易请求先提交到validator，validator 通过Gossip 协议，类似P2P 协议互相交换信息，确保交易能找到当前slot 对应的leader，某个slot 期间只有一个leader， 邻近 validator 会把交易转发给当前leader 以及未来leader.
2. 当前slot 对应的leader 是在epoch 开始的时候事先计算出来的
3. Solana，交易一般不会像以太坊那样广播给全网所有节点（即没有全局 Mempool），而是通过 Gossip 协议尽可能在部分节点中传播，但最终必须送达到当前 Leader 节点，才能被打包进区块。其他节点收到交易之后会尽可能把交易发往leader, 只有leader 能打包区块. 

* leader 挂掉
下一个slot leader 接着打包, 交易不会丢失会存于网络中的其他节点

## leader 生成新区块之后如何广播 ，具体 ？
1. 使用turbine 协议，分片广播，让部分节点转发区块，减少带宽负担，避免全网广播

# POH
<img width="795" alt="image" src="https://github.com/user-attachments/assets/a008657b-4d3f-4f2b-a05f-c3d176e3fc28" />

POH 是一个链式结构，前面hash 的输出是后面hash 的输入，只要hash 是防碰撞的，为了得到后续的hash 结果必须依次从头计算，而且只能单线程计算，所以前序
事件肯定发生在后续事件的前面，并且可以根据hash 的计算时间以及区块的生成间隔，大致估算事件发生的时间

## POH 的验证
<img width="596" alt="image" src="https://github.com/user-attachments/assets/0e92b718-ac8b-48bf-9867-938c08caf7ae" />
因为当验证时已经获得了每个index 的hash output，只需要根据input 和data，计算是否output 是符合预期的，所以在多核cpu 上可以并行运行，验证的时间会比生成POH 链的时间大幅降低。 


# 为什么solana 交易这么快
1. POH 的机制下, 避免了交易排序的共识时间, eth 下的交易排序需要共识, 交易插入POH 链即有了顺序.
2. 没有类似eth 的mempool, 维护一个全局共识的mempool 虽然增加了交易的透明性也会增加延迟, solana 的交易直接通过p2p 网络发往leader.
3. 校验POH 链很快, 可以并行去做, 因为已经知道每个POH 节点的输入和输出了, 单线程生成POH 多线程验证
4. Solana 的 Sealevel 引擎允许 多个交易同时执行，如果它们不冲突（即不同时操作同一账户）; 基于账户依赖图分析，自动调度交易执行顺序; 相比以太坊单线程顺序执行，极大提升并发。

# why fee in solana is low

# global clock sync
因为400 ms 一个block time , 在leader schedule 中, 下一个指定的leader 如何在上一个leader 生成一个新的block 之后, 仅过了不到半秒, 就继续接着生成下一个block 呢, 这个全局的精确时钟同步是如何做到的?

内部有一个sha256 counter, 由于sha256 函数在所有电脑芯片上跑几乎都差不多时间, 所以根据每台电脑上sha256 的计算次数来大概的预估时间. 

计算公式:
hash(n)= sha256(hash(n - 1), counter)

每个validator 都会在本地单核单线程的运行这个公式, 由于在不同电脑运行的时间都相差无几, counter 几乎都一样, 
通过事先计算好的leader schedule list, 各个未来的leader 知道各自要运行的slot , 把这个slot 转换成未来的
counter 就可以按时当上leader 

# 如何动态调整POH 的生成速度

# 如何选择最长链, 如果有某个slot 出块延迟, 如何处理
会选择最大的 POH counter 作为最长链, 例如
Slot 1：Leader A 未在规定时间内出块，网络超时。

Slot 2：Leader B 生成区块 2（PoH 计数器 0 - 15000）。

Slot 3：Leader C 生成区块 3（PoH 计数器 15001 - 30000）。

此时网络中存在两条链：

链 1：区块 0 → 区块 2 → 区块 3（总 PoH 计数器增量 30000）

链 2：区块 0 → 区块 1（假设 Leader A 最终出块，PoH 计数器 0 - 10000）

由于链 1 的 PoH 计数器增量更大，节点会选择链 1 作为主链，丢弃链 2

* 某个slot 出块延迟

block 就算后续超时出来会被丢弃, 选择POH 计数更大的作为最终链, 由于交易是广播到所有节点的内存池中, 发给leader a 的交易没有被纳入block , 但是也发送给了其他leader, 
其他leader 判断交易如果不在最近的区块交易中, 那么就会把该交易纳入区块, 因为最终肯定是有一条链能胜出, 所以最终链如果没有包含该交易, 那么交易是肯定会被纳入新区块的.

# 为什么gas fee 比 eth 要低

Solana它的交易费用是根据交易的复杂度和大小动态计算的，这意味着，交易费用会根据交易的执行成本而变化，而不是根据网络上的交易量变化。Solana 的平均交易费用通常低于 0.01 美元，平均为 0.00025 美元，这使得进行小额交易更具成本效益。

以太坊 的交易费用因网络拥堵而波动，这是一种纯粹的市场机制，网络中交易拥堵情况下你的交易要想被确认，就需要支付高昂的手续费。目前一笔转账交易的手续费大概在 1 ～ 10 美元左右。


# 如何指定 Leader 会考虑诸多因素

1.质押的代币数量： 在 PoS 中，质押的代币数量是一个关键的考虑因素。Validator 通常倾向于选择质押数量较大的节点，因为这增加了节点被选中为区块生产者的机会。这也有助于确保网络由具有足够利益参与的节点维护。

2.节点的性能： Validator 的节点性能是另一个关键因素。高性能的节点能够更快速地验证和打包交易，有助于维持网络的高吞吐量。Validator 可能会选择性能较好的节点以提高整个网络的效率。

3.网络延迟： Validator 可能会考虑节点之间的网络延迟。选择网络延迟较低的节点有助于减少区块的传播时间，从而提高网络的实时性。

4.节点的可用性： Validator 会关注节点的可用性，确保它们能够稳定运行而不容易出现故障。可靠的节点能够为网络提供更稳定的服务。

 

# 去中心化考虑

虽然 Solana 在任何时刻只有一个 leader，但这并不意味着系统是中心化的。以下几点说明了 Solana 如何维护去中心化：

频繁轮换:
Leader 角色快速轮换（每 400 毫秒）大大降低了单一实体控制网络的风险。

基于质押的概率选择:
成为 leader 的机会与验证者的质押量成正比，鼓励更多参与者加入网络。

大量验证者:
Solana 网络有数百个活跃验证者，确保了足够的去中心化程度。

验证者的持续参与:
即使不是 leader，其他验证者也在持续验证交易和区块，维护网络安全。

惩罚机制:
对于行为不当的验证者（包括 leader），存在削减质押的惩罚机制。

开放参与:
任何人只要满足硬件要求并质押足够的 SOL，都可以成为验证者。

治理决策:
网络参数和升级通过去中心化的治理过程决定，而不是由单一实体控制

# PDA

在Solana区块链中，PDA指的是“程序派生地址”（Program Derived Address）。这是一种特殊类型的地址，由 Solana 的程序生成，而不是由用户的私钥直接派生。PDA的主要目的是允许程序拥有和控制某些数据或资产，而不需要传统的私钥签名。

# solana 如何保证交易能买到meme token, 如何设置computer unit
1. 查询链上meme token, CU 的分布, 得到通常的出价情况, 因为设置了CU 的指令是能在交易历史中看到的
2. 然后给用户几种选项, 例如90 分位值大概率能抢到, 这个价位是否划算由用户自行判断. 

# Solana交易的确认级别
1. (最近的confirmed block 和 最近finalized的block 差了32 个slot, 大约13 秒) The tradeoff is that there is typically at least a 32 slot difference between the most recent confirmed block and the most recent finalized block. This tradeoff is pretty severe and effectively reduces the expiration of your transactions by about 13 seconds but this could be even more during unstable cluster conditions.
2. processed 的区块可能有百分之五不会变为 finalized. 


## 为什么需要 PDA？

在区块链中，你需要一个私钥来证明你拥有一个公钥的所有权，同时你才能签字同意这个账户的转账请求。但如果这个账户的所有者不是一个人而是一个去中心化程序，那么把私钥放在这个程序上就不是一个好主意，因为所有程序代码都在链上都是公开的，如果所有人都能看到你的私钥，那么人们就能进行一些恶意操作，比如偷走你的代币。这时我们就需要一个没有私钥的 PDA。 这样程序不需要私钥就能对一个地址进行签名操作。
