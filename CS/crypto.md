# TODO
1. 究竟数字签名如何促成交易.

# 钱包
## 私钥, 公钥, 地址
通过一些手段例如助记词生成用户的私钥, 用私钥签名交易后, 把交易信息发到链上, 使用椭圆曲线加密算法（如ECDSA）从私钥生成公钥。对公钥进行哈希处理（不同的加密货币有不同的哈希算法和编码格式）生成用户地址

* 地址冲突

有可能的, 但是几率很低, SHA-256：输出是256位（32字节），即有2^256种可能的哈希值。即使发生了地址碰撞, 没有私钥也是无法访问地址的

* 私钥冲突

比特币私钥是一个256位的随机数。私钥空间有2^256种可能性，大约等于10^77, 每种可能性出现的概率相同, 所以冲突概率几乎等于零
## 智能合约钱包
### feature
* Smart contract code is public: Anyone can view, audit and verify the smart contract code.
* Blockchain data transparency: All transactions and operations are recorded on the blockchain, open and transparent.
* Immutable ledger: Blockchain technology ensures the immutability of records.
* Community auditing and verification: Smart contracts are usually audited by the community and professional auditing companies to ensure their security and correctness.

## 中心化交易所的钱包如何保证安全
Tiered storage architecture
### Hot wallet layer
handles daily transactions and user withdrawal requests, and has the ability to respond quickly.
### Cold wallet layer 
stores most of the funds, keeps them offline, and only interacts with the hot wallet when large transfers are required.

### Multi-signature control

Interaction between cold and hot wallets requires multi-signature authorization to prevent single points of failure and internal abuse.

### Deposit

 Users deposit cryptocurrencies to the address provided by Binance, with funds going into the hot wallet first. Hot wallets regularly transfer funds beyond daily needs to cold wallets.
### Withdrawal
 When a user initiates a withdrawal request, funds are transferred from the hot wallet. If the hot wallet balance is insufficient, the system will initiate the process of transferring funds from the cold wallet to the hot wallet
# 交易费用, gas
### 为什么需要gas
#### Prevent Network Abuse
Gas fees ensure every action costs something, limiting how much computing power each user can use. And prevent DDOS attack.
#### Miner Rewards
Motivate Miners to Process Transactions
#### Encourage Efficient Code
 Developers aim to write optimized code to use less gas, making smart contracts cheaper and more efficient for users.

#### Build Economic Models
 Gas fees help create an economic system within the blockchain, ensuring the network can sustain itself through fees.
# network
## 如何发现其他节点, 服务发现的
* 通过dns 动态获取对应ip, 如果获取超时, 则直连在代码中写死的一批长时间保活的服务器. IBD 节点如果和任意节点连上也就获得了该节点知晓的所有节点, 由此可见p2p 架构的优势. 
* 那么如何鼓励总是有节点在网络中呢? 应该是有利益驱使. 

# block chain
每个block 包含若干交易, 整个block 与其数据的hash 和之前block 的hash 一起形成一个block header hash(如何形成? ), 所以等于说其中的一个交易一旦被修改, 后续的所有block 的hash 都变了. 

The merkle root is stored in the block header. Each block also stores the hash of the previous block’s header, chaining the blocks together. This ensures a transaction cannot be modified without modifying the block that records it and all following blocks.

Transactions are also chained together. Bitcoin wallet software gives the impression that satoshis are sent from and to wallets, but bitcoins really move from transaction to transaction. Each transaction spends the satoshis previously received in one or more earlier transactions, so the input of one transaction is the output of a previous transaction.

交易费用的由来, input 大于output 的部分: but if the inputs exceed the value of the outputs, any difference in value may be claimed as a transaction fee by the Bitcoin miner who creates the block containing that transaction. 

## 工作量证明
没看懂...
代码中可以控制hash 合法的难度, 太难了降低, 太容易了升高, 用生成固定数量的区块的花的时间来衡量. 而如何叫做合法的hash, 可以理解为小于某个特定值的hash. 概率是已知的.

## Block Height And Forking
同时生成多个block 的时候, 并发时如何连接block 的问题? 

## Transaction Data
Merkle root 如何构成, 交易的hash 两两hash 而成, 

## coinbase 交易
什么是 Coinbase 交易？
> 比特币区块链上的每个区块之中都会包含一个或者多个交易（ transaction ），其中第一个交易就叫做 Coinbase 交易。Coinbase 交易是矿工创建的，主要是为了奖励矿工为了进行 POW 挖矿而付出的努力。

奖励分为两部分。一部分是出块奖励，这部分是相对固定的，当前每个区块的出块奖励是12.5BTC，每四年减半一次。另外一部分是手续费，当前区块的每个交易中都会包含一定的对矿工的奖励，也就是交易手续费。创建 Coinbase 交易的时候，矿工会把所有交易中的手续费累加到一起，然后把这笔钱转账给自己。

> Coinbase 交易的特点是没有“父交易”。普通交易中需要 input ，而 input 是来自父交易的 output ，所以普通交易是有父交易的。但是 Coinbase 交易是没有父交易的，因为币是直接由系统生成的。

什么是 coinbase ？
那么 coinbase 交易中的 coinbase 这个词是什么意思呢？

简单来说 coinbase 就是系统生成的币。“Coinbase 交易”也叫做 “Generation 交易”，也就是“生成交易”，这是因为其他的普通交易中，都是去转账已有的比特币，而这个交易是专门从无到有的去生成新的比特币的。精确一点说，coinbase 就是“生成交易”中的 input 。

这就是 coinbase 这个词的基本含义。

Coinbase 交易中包含的数据
下面来仔细聊聊 coinbase 交易中包含的各项数据。

交易中包含一个 input 和一个 output 。这个 input 就是 coinbase 。output 指向矿工的地址，总金额等于 coinbase 加上区块中全部交易的手续费。

另外 coinbase 中还有一个最多100字节的数据。除了最开始的几个字节，这个数据中剩下的地方可以存储任意数据。矿工可以用来存储任何自己想要存储的数据。例如在创世区块，也就是 genesis block 中，中本聪保存了这样一句话：

The Times 03/Jan/2009 Chancellor on brink of second bailout for banks

数据的最开始几个字节保存的是区块高度。所谓区块高度就是当前区块跟创世区块之间间隔的区块数量。创世区块就是比特币区块链上的第一个区块，区块高度是零，依次类推。

## 挖矿

应该就是交易生成之后广播, 然后第一个算出合法hash 的机器把这个区块广播, 最后output - input 之差作为交易费用, 那么多少个交易才构成一个区块呢? 
coinbase transaction 是什么? 
As illustrated below, solo miners typically use bitcoind to get new transactions from the network. Their mining software periodically polls bitcoind for new transactions using the “getblocktemplate” RPC, which provides the list of new transactions plus the public key to which the coinbase transaction should be sent.

The mining software constructs a block using the template (described below) and creates a block header. It then sends the 80-byte block header to its mining hardware (an ASIC) along with a target threshold (difficulty setting). The mining hardware iterates through every possible value for the block header nonce and generates the corresponding hash. (iterates through every possible value 只能通过遍历来获取符合threshold 的hash )