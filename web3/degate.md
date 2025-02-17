

# How did you research on DeGate?

1. 先总览一遍文档, 有个大体的了解, 不懂的问题记录下来, 最后去找答案, 有必要就看白皮书, 白皮书能反映当时创立的想法, 但可能不完全符合现在的情况, 但是最好有时间也能读一下.

2. 用了一些常用的功能, 包括充值和买卖, 目前充值只能直接充值usdt, usdc, eth, 最好能多支持几种常用的token, 例如sol, 会更方便

3. 了解一些最重要功能背后的链路, 例如从充值到撮合订单, 到促成交易, 了解清楚后其他功能的逻辑可能就大同小异

4. 了解degate 的优劣 

# In summary, what do you understand about DeGate?

1. first is the summary:
A fairly launched, Dao-centric, Zero Knowledge based self-custody trading protocol built on Ethereum. DeGate is Limit Orders, Decentralized and Self-Custody.

DeGate is an Orderbook Decentralized Exchange (DEX) protocol built on Zero Knowledge (ZK) technology. As a ZK Rollup, DeGate fills a key gap in the market by providing spot order book trading in self-custody manner, and grid trading within the Ethereum ecosystem, offering an experience similar to centralized exchanges (CEX). DeGate is a DAO-centric, self-custody exchange, with a DAO fully controlling its treasury. DeGate is a protocol of the community, by the community, and for the community.


2. <img width="1512" alt="image" src="https://github.com/user-attachments/assets/bc44e0c9-3547-4eca-a5c4-97fe87d3ce57" />


3. 拿用户充值来举例子
3.1 用户发起转账到degate 的合约, 不会直接在eth 链上发起一笔交易, 由于degate 在layer2 , 会把请求发给degate off chain node, offchain node 会发起一笔交易, 提交到zk roll up service , 这个serivce 由operator, circuit, postman , merkle tree 等module 组成, 由于这是一个infra service, 暂时不会深入他的细节, 只管他的输入和输出, 输入就是一个off chain transaction, 多个transaction会生成一个 zkBlocks, and generate the proof for the zkblock, 把这个proof 提交到链上来更新链上的状态, 链上部分由多个合约组成, 会协调处理链上请求, 例如storing user funds, verifying the zero-knowledge proofs submitted by the off-chain node and storing the latest Merkle tree roots.

4. 特点

4.1 比传统的dex 节省gas fee 和交易快速, 也比直接在l1 网络交易要快, 因为 only sends packaged information on-chain for zero-knowledge proof, 相当于在链下把交易打包之后发送到链上, 减少了链上的计算 which is different from traditional DEXs where every order and transaction is initiated and completed on-chain. This saves a lot of gas fees, and does not depend on the confirmation time of the L1 blockchain network, 这会提升交易的qps

4.2  trustless, transparency
这两个比较类似打算一起来说, 因为是dex, 代码开源, 可以供查看和audit, 保证了透明性, 增加了信任度; 

4.3 self custody of assets,
DeGate operates as a DAO, which has full control over the Treasury. The members of the DAO are DG token holders, who participate in the governance process, and delegate decision making. 
also provide “Exodus Mode ”, 保证用户的钱在极端offchain 服务挂掉的情况下也能取得出来; 

4.4 data avaliablity
主要依赖于l1 的eth, 因为打包后的交易信息存储在eth, 如果链下的degate node 挂了, 也有 Exodus Mode 保证钱能去出来

4.5 其他功能
grid trading (适用于半自动化的快速交易)
Permissionless Listing(任何人都可以自由到degate上币, 但可能缺乏流动性)

5. 个人认为需要提高的点, 
5.1 虽然我们强调是去中心化的, 由dao 管理, 我目前并没有找到详细资料关于如何参与DAO 以及DAO 是如何按照schedule 运作的, 可能我漏掉了,   
而且虽然我们代码已经开源, 但多数人不太可能通过代码去构建对我们平台的信任, degate 的关键点在于链下打包交易和零知识证明, 零知识证明是关键, 但是比较晦涩难懂, 关于零知识证明没有找到一个通俗易懂的描述, 应该着重强调如何能让用户信任, 是否能做到足够安全; 关于链下处理交易也缺乏详细的描述

5.2 对比其他交易量更大的DEX, 例如raydium on solana 
由于solana 的POH 和Sealevel 等技术, 在交易延迟, 交易吞吐量, 交易费用方面据我了解可能都比degate 更具吸引力, 而且所有操作都在链上, 更容易被用户相信, 所以我认为degate 需要重点开发跨链部分, 支持交易任何链和任何token , 由于我们off chain 处理的灵活性, 这部分有天然的优势, 需要加以放大; 在交易费用方面也尽可能的进一步优化. 



# How does DeGate project align as an opportunity for your Professional and Personal goals?

因为我认为后续web3 还是交易的天下, 从一开始的bitcoin, 再到eth 上的智能合约, 再到defi, NFT 和现在的meme token , 无一不证明了交易一直是主赛道, 而DeGate is an Orderbook Decentralized Exchange (DEX) protocol built on Zero Knowledge (ZK) technology, 我能学习到很多交易方面的知识, 以及l1 和 l2 网络的在交易方面的trade off, 既符合个人的兴趣又能赶上发展的趋势. 

### trade fee
Trading Fee Rate of Order Book Pairs
Stable trading pairs: Maker 0.00%, Taker 0.01%
Other trading pairs: Maker 0.00%, Taker 0.07% 0.1%

Trading Fee Rate of Swap Pairs
0.99%, on top of external DEX fee rate

Gas Fee
Gas fees are calculated based on the real-time gas costs on Ethereum.


## Fast transaction
链下将交易批量打包, 只发送打包信息到链上做零知识证明, 不需要等待链上确认

## Trustless

## Data availablty

## Transparency

## Orderbook transaction


# Feature



## Grid Trading

The grid trading function is another innovation of the DeGate protocol. This feature replicates the grid trading on a CEX, enabling users to implement a trading strategy based on the ups and downs in a trading pair while maintaining full self-custody of their assets.

什么是网格交易

 when a buy order is executed, it instantly places another sell order at a higher grid level, and vice versa. Grid strategy performs best in sideways markets when prices fluctuate regularly within a defined range, enabling you to make profits on small price changes.

##  permissionless free token listing

支持任何人都有权限上币, 如何做到, 其他dex 为什么不行
其他的: The current method of determining whether a new trading pair can be listed on a DEX is to decide through DAO governance or admin authority.

## Decentralized and Self-Custody, support Limit Orders
A fairly launched, Dao-centric, Zero Knowledge based self-custody trading protocol built on Ethereum. DeGate is Limit Orders, Decentralized and Self-Custody.

DeGate is an Orderbook Decentralized Exchange (DEX) protocol built on Zero Knowledge (ZK) technology. As a ZK Rollup, DeGate fills a key gap in the market by providing spot order book trading in self-custody manner, and grid trading within the Ethereum ecosystem, offering an experience similar to centralized exchanges (CEX). DeGate is a DAO-centric, self-custody exchange, with a DAO fully controlling its treasury. DeGate is a protocol of the community, by the community, and for the community.

##  trust mechanism of the entire DeGate protocol
the is guaranteed by two factors: ZK-Rollup data availability and "Exodus Mode".

### zk roll-up

it increases scalability by processing a large number of transactions and "rolling" them into a transaction on a base layer, greatly reducing cost.

In the ZK-Rollup technical architecture, the protocol must host users' assets in smart contracts

### Exodus Mode
保证钱在服务挂掉的极端情况下能取出来. 

To illustrate, suppose any force withdrawal has not been processed for more than 15 days. Anyone can trigger a transaction to enable the exodus mode of DeGate protocol. This process is irreversible, which means that this deployment instance of the DeGate protocol is no longer useable. The only thing that can be done is for users to withdraw assets by directly providing data proof as a parameter to the smart contract. Users can restore their data from historical call data stored on Ethereum.



## Gas fee less
Gas fees are a major concern on the Ethereum mainnet. Conventional AMM DEXes incur high gas fees on Ethereum and provide only market orders, where traders have to accept the current market price for a trading pair. 
通过zk 解决了其他AMM dex gas fee 高, 且只能market order的问题, 提供了limit order. 
为什么呢?

### 如何节省gas fee
Gas Saving Deposit: Depositing into a DEX protocol often incurs a high one-time fee. DeGate has created a gas-saving deposit option. This option is based on a “simple transfer” rather than a “contract call”. This method can reduce the one-time gas deposit fee by up to 75%.

通过类似转账而不是合约调用, 为什么合约调用有更高gas 



