# Goal
The intent of Ethereum is to create an alternative protocol for building decentralized applications, providing a different set of tradeoffs that we believe will be very useful for a large class of decentralized applications, with particular emphasis on situations where rapid development time, security for small and rarely used applications, and the ability of different applications to very efficiently interact, are important.

# POS 的演进
## chain based proof of stake
![image](https://github.com/user-attachments/assets/ddf55967-5676-43a4-86e6-4ce22f09fdf2)
### 问题
* 随机种子难以真正随机

很难找到一个完全真正随机的随机种子 r, 因为受prevblock 的影响, 假设prevblock 是我出块, 我可以调整交易的顺序, 穷举, 使得r 计算出来下一个proposer 还是我. 可以用事先计算的方式, 都算好, 避免受上一个区块的影响. 
* 其他问题
![image](https://github.com/user-attachments/assets/b8085a11-e435-402f-9574-0da16ca62b2b)

## BFT based POS
### goal
能够正常的出块, 所有诚实节点达成一致, 表现好像一个中心化系统, 
![image](https://github.com/user-attachments/assets/5d8ea96a-2441-4306-be84-30310b262b30)

![image](https://github.com/user-attachments/assets/f8e8d8e9-7c2d-4082-a024-a1da9b01ce40)

### 如何达成共识
![image](https://github.com/user-attachments/assets/a69c732f-60b1-486a-b28b-dfcf0b8a3397)
多轮投票让三分之二的节点都同意

* 发生异常

会由下一轮的proposer 再去投票, 每轮的proposer 是事先由block 高度选出来的, 如果没有异常就很快走绿色, 达到三分之二即可达成一致. 
![image](https://github.com/user-attachments/assets/88bd9de1-7415-4f7a-a3f5-667639e3fdb0)

# eth mempool
## 为什么需要全局 Mempool
去中心化交易广播: 不依赖特定验证者，任何交易可被全网任意节点接收和广播
防止交易审查: 避免交易只能提交给单一验证者导致交易被拒绝或审查
市场定价机制: Mempool 公开，Gas 费竞争公开，市场自由竞价，防止暗箱操作
公平交易排序: 公开交易，机器人（如 MEV bot）、用户都能看到，促进公平参与（虽然引发抢跑）
缓冲区: 当区块拥堵时，交易排队缓冲，避免用户交易丢失
## 全局 Mempool 是如何实现的
1. 节点接收交易
用户将交易发送给任意以太坊节点（通过 JSON-RPC）。

2. 节点 Gossip 广播
节点收到交易后，通过 Gossip Protocol 将交易广播给其他邻居节点。
最终达到全网同步，所有节点 mempool 都包含该交易。
3. 区块提议者从 mempool 取交易
下一轮出块的 proposer (验证者/矿工) 从 mempool 中选择交易打包。
4. 有去重机制，避免无限广播。

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



