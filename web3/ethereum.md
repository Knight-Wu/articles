# Goal
The intent of Ethereum is to create an alternative protocol for building decentralized applications, providing a different set of tradeoffs that we believe will be very useful for a large class of decentralized applications, with particular emphasis on situations where rapid development time, security for small and rarely used applications, and the ability of different applications to very efficiently interact, are important.

# Eth account
Ethereum account contains four fields:

The nonce, a counter used to make sure each transaction can only be processed once
The account's current ether balance
The account's contract code, if present
The account's storage (empty by default)
</br>
"Ether" is the main internal crypto-fuel of Ethereum, and is used to pay transaction fees. In general, there are two types of accounts: externally owned accounts, controlled by private keys, and contract accounts, controlled by their contract code. An externally owned account has no code, and one can send messages from an externally owned account by creating and signing a transaction; in a contract account, every time the contract account receives a message its code activates, allowing it to read and write to internal storage and send other messages or create contracts in turn.

## account nonce
每个以太坊账户（地址）都有一个独立的 Nonce，表示该账户已发送的交易次数。
核心作用：

交易顺序控制：
每笔交易必须包含一个 Nonce 值，且必须严格等于账户当前的 Nonce（初始为 0）。
例如：

第1笔交易 Nonce = 0

第2笔交易 Nonce = 1

依此类推...
节点会拒绝 Nonce 不连续或重复的交易，确保交易按顺序处理。

防止双花（重复交易）：
如果用户试图发送两笔 Nonce 相同的交易，只有第一笔会被矿工打包，后续交易会被视为无效。

交易替换：
如果你想加速一笔未确认的低 Gas 费交易，可以重新发送一笔 相同 Nonce 但 Gas 费更高的交易，矿工会优先处理后者。

# ETH Layer 2
## Type of layer 2s
L2s come in different shapes and sizes in terms of their relationship with Ethereum: Each design decision comes with trade-offs in terms of security, scalability, or decentralization.
取决于去中心化, 安全性, 可扩展性的tradeoff, 具体说来就是把什么数据或者计算量放到 L1 上.
## Polygon CDK 
### arch
![image](https://github.com/user-attachments/assets/22f7a0ee-2c04-45ca-b0ed-be2b36a8d7a5)
https://docs.polygon.technology/cdk/concepts/architecture/ 
## Data availably
### Validium
Validiums achieve scalability by keeping all transaction data off-chain and only post state commitments (and validity proofs) when relaying state updates to the main Ethereum chain. While off-chain data availability introduces trade-offs, it can lead to massive improvements in scalability (validiums can process ~9,000 transactions).

* VS other layer2 project

optimistic rollups and ZK-rollups 把一些交易数据存到链上, 但是他们的扩展性就会收到eth 主网带宽的限制.

### Transaction lifecycle
Submitted: The transaction is submitted to the L2.
Executed: The transaction is executed on the L2 by the sequencer.(对于用户来说已经完成了)
Batched: The transaction is included in a batch of transactions.
Sequenced: The batch containing the transaction is sent to Ethereum.
Aggregated: A ZK-proof is generated, posted, and verified on Ethereum to prove the transaction is valid.

* transaction finality
  ![image](https://github.com/user-attachments/assets/79abc639-104f-4975-a0f8-def0f935ce80)

### Batch, block, Transaction
![image](https://github.com/user-attachments/assets/0ef3a3e3-cbf0-4178-adb4-5ccd1c4f2651)

## xlayer
### Architectural
![image](https://github.com/user-attachments/assets/5465ae04-9e8d-4d13-9b5c-8b50927ea4e7)
