# TODO
1. 究竟数字签名如何促成交易.

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

## 挖矿
应该就是交易生成之后广播, 然后第一个算出合法hash 的机器把这个区块广播, 最后output - input 之差作为交易费用, 那么多少个交易才构成一个区块呢? 
coinbase transaction 是什么? 
As illustrated below, solo miners typically use bitcoind to get new transactions from the network. Their mining software periodically polls bitcoind for new transactions using the “getblocktemplate” RPC, which provides the list of new transactions plus the public key to which the coinbase transaction should be sent.

The mining software constructs a block using the template (described below) and creates a block header. It then sends the 80-byte block header to its mining hardware (an ASIC) along with a target threshold (difficulty setting). The mining hardware iterates through every possible value for the block header nonce and generates the corresponding hash. (iterates through every possible value 什么意思? )