<img width="596" alt="image" src="https://github.com/user-attachments/assets/9952c4b2-97f2-4661-aeb2-f3f5a19291a8" /># solana 如何判断某个交易是否过期，过期时间是多少
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
2. 每个slot（时间单位，400MS，bitcoin 是十分钟，eth 是12秒）会有一个leader，每个epoch 周期由多个slot 组成，每次开始新的epoch 会事先计算好每个slot 对应的leader，

## 为什么需要有epoch 
一个epoch 大约为2.5 天，每次质押的sol 变化将在下个epoch 生效，每次epoch 结束时会清理不活跃的账户并收取租金。

## 选出leader 之后
1. 接受交易请求，并将交易打包成区块，计算新的POH 链
2. 将新的区块广播给其他validator 验证
3. validator 通过tower BFT 进行共识，多数质押权重的validator 认可生成的新的POH 链，则该区块被确认；
4. 如果链产生了多个分叉，哪个分叉得到的投票权重高则保留。

## 交易请求如何发送到leader
1. 某个slot 期间只有一个leader， 如果leader 没有及时生成新的区块，等到slot 超时（400ms）会转发到新的leader
2. 交易请求先提交到validator， validator 通过Gossip 协议，类似P2P 协议互相交换信息，确保交易能找到当前slot 对应的leader，
当前slot 对应的leader 是在epoch 开始的时候事先计算出来的

## leader 生成新区块之后如何广播 ，具体 ？
1. 使用turbine 协议，分片广播，让部分节点转发区块，减少带宽负担，避免全网广播

# POH
<img width="795" alt="image" src="https://github.com/user-attachments/assets/a008657b-4d3f-4f2b-a05f-c3d176e3fc28" />

POH 是一个链式结构，前面hash 的输出是后面hash 的输入，只要hash 是防碰撞的，为了得到后续的hash 结果必须依次从头计算，而且只能单线程计算，所以前序
事件肯定发生在后续事件的前面，并且可以根据hash 的计算时间以及区块的生成间隔，大致估算事件发生的时间

## POH 的验证
<img width="596" alt="image" src="https://github.com/user-attachments/assets/0e92b718-ac8b-48bf-9867-938c08caf7ae" />
因为当验证时已经获得了每个index 的hash output，只需要根据input 和data，计算是否output 是符合预期的，所以在多核cpu 上可以并行运行，验证的时间会比生成POH 链的时间大幅降低。 


