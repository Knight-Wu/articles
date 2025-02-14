# what is orderly network
他提供了基于订单铺模型的交易基础架构, 支持全链, 用户可以使用提供的sdk 来构建一个支持多种交易的平台, 并且交易可以跨链, 不像传统跨链桥那么繁琐且不灵活

# How to implement cross chain trade
概括来说, 在不同链的用户把当创建跨链order 时, 通过各自链的asset layer 去转发和接受请求, engine layer 通过撮合引擎匹配跨链订单, 把订单数据保存在settlement layer,  并周期性的把交易数据打包发送给eth, settlement layer 把请求转发到assert layer 用于更新用户余额, 各个layer 的沟通都是通过LayerZero.

## Asset Layer (Asset Vaults) resides on each chain Orderly supports

Users interact with this layer when registering/depositing/withdrawing and where the funds reside.

## Settlement Layer (Orderly L2) - resides on a single chain

Users do not interact with this directly and is used as a transaction ledger for storing transaction/user data.

## Engine Layer (Orderbook) - Orderbook and order-related services

Users interact with this when managing orders. This where trade execution happens and includes matching engine itself, risk engine, and other services.
Orders from different chains come into the same orderbook, making it chain agnostic unifying liquidity unlike multichain DEXes. After the orders are matched, they are uploaded and settled on Orderly L2 chain built using OP Stack, settling to Ethereum peridically.

With all orders being settled on-chain, Orderly’s settlement layer sends/receives deposit/withdrawal instructions from the engine layer and updates users’ on-chain balance. The settlement layer then relays these instructions to/from the Asset Vaults — all of this communication is powered by LayerZero.