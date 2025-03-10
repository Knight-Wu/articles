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
## Data availability
### Validium
Validiums achieve scalability by keeping all transaction data off-chain and only post state commitments (and validity proofs) when relaying state updates to the main Ethereum chain. While off-chain data availability introduces trade-offs, it can lead to massive improvements in scalability (validiums can process ~9,000 transactions).

* VS other layer2 project

optimistic rollups and ZK-rollups 把一些交易数据存到链上, 但是他们的扩展性就会收到eth 主网带宽的限制.
#### Data availablity committee (DAC)
* First of all, the transaction data is not published to the L1 but only the hash of the data.
* Secondly, a trusted-sequencer collects transactions from the transaction pool manager, puts them into batches and computes the hash of the transaction data.
* After verifying the proposed hash values individually, each DAC member signs them and sends the signature to the sequencer.(tx hash 需要被DAC member 验证)

#### Data flow
![image](https://github.com/user-attachments/assets/0288dc6d-952c-4853-a911-a3f090311e3a)
1. L2 sequencer  收集用户交易, 并分成不同batch, 计算batch hash 
2. 将batch 数据和hash 发送给DAC 做验证, DAC 验证之后做签名, 并存储hash 在本地数据库
3. Sequencer 将DAC 的签名和hash 发送到ETH 上, 等待验证.
4. ETH 上使用多签技术, 验证DAC 的签名是否有效, 进而确定这个batch 是否得到批准.
5. aggregator 生成prove 给prover 做验证. 
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

### ZK prove
* how it works

As we already mentioned, there is a prover (the party that proves that they have some information) and a verifier (the party that verifies the prover has the info).

在这个过程的第一步中，证明者（Alice）和验证者（Bob）先确定使用哪些参数和加密算法。

然后，证明者生成一个加密承诺(commitment)，这个承诺代表了她所证明的陈述内容，但并不暴露具体内容。

接着，验证者会随机提出挑战(challenge)，证明者根据挑战和承诺生成一个回应(response)。

然后，验证者会将回应与挑战和承诺进行对比，来判断陈述是否有效。

从挑战开始，验证者可以多次提出挑战，以确保陈述是正确的，并且置信度越来越高。

简而言之，这个零知识证明过程主要包含3个步骤：承诺、挑战和回应。

举个例子，假设我们说的是世界上最好吃的巧克力饼干，这个过程可以这么描述：

Bob和Alice约定，要证明她有这个食谱，Alice会烤饼干，而Bob会尝尝这些饼干（尝饼干就是挑战）。Alice烤好了饼干，Bob尝了一口，结果饼干确实是世界上最好吃的巧克力饼干。
## xlayer
### Architectural
![image](https://github.com/user-attachments/assets/5465ae04-9e8d-4d13-9b5c-8b50927ea4e7)


# uniswap
## uniswap v3 
### 集中流动性增加了无偿损失(Impermanent Loss, IL)
集中流动性提升了资金使用率, 提升了收益, 与此同时会增加风险, 当价格越过用户设置的价格区间时会进一步放大无偿损失. 

因为无偿损失出现在一个token 价格剧烈波动的时候, 因为x*y = k, x , y 数量呈比例变化, 但是当某个token 价格剧烈变化时, 但是另一个token 价格不变, 那么必然会产生价值的波动, 因为 number x * price x + number y * price y = total value, 
只要某个token 价格出现变化, 不管是增加还是降低都会引起IL, 

* eth, usdt, eth 价格增加

![image](https://github.com/user-attachments/assets/e3fb47a1-213d-4592-bf83-3bafcfa18d8d)

* eth, usdt, eth 价格降低

![image](https://github.com/user-attachments/assets/323d9a35-d411-4738-a20e-fc7e18424547)



