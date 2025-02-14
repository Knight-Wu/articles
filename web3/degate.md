# How to research degate
先总览一遍文档, 有个大体的了解, 不懂的问题记录下来, 最后去找答案, 有必要就看白皮书, 但是白皮书是之前的设想, 可能不符合当前的情况了



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




# Zero Knowledge Rollup technology
on-chain and off-chain operations that enable off-chain processing of all account and asset changes followed by a rollup to the on-chain smart contract.
