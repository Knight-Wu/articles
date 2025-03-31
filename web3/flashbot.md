# MEV 行为好处
1. 帮助dex 运行, 触发链上的程序

# 坏处
1. 有时会极大拉升gas 费, 提升交易成本
2. 低gas 的交易无法成交, 失败交易上链, 浪费空间
3. 进入公开mempool 之后容易被攻击, 提取价值
4. miner 为了MEV 进行reorg

# EIP-1559
EIP-1559 是以太坊于 2021 年 伦敦硬分叉中引入的一项协议变更，目的是：

使交易费用更可预测

减少抢 Gas 的现象（Gas War）

抑制 MEV 乱象

实现部分 ETH 销毁（让 ETH 变得更稀缺）
* gas fee
Total Fee = Base Fee + Priority Fee (Tip)
</br>

* base fee
Base Fee 是协议自动设定的 每单位 Gas 的最小价格, 不由用户设定，也不会给到矿工，而是 销毁（burn）掉, 体现的是网络拥堵程度

默认 目标值为 15,000,000 gas

最大允许容量为 30,000,000 gas

* 拥堵判断逻辑
  
用 前一个区块的 Gas 使用量 与 目标值 做对比, 默认 目标值为 15,000,000 gas, 最大允许容量为 30,000,000 gas, 如果链上交易活跃、需求高 , 区块会接近 30M

如果链上交易少 , 区块可能远低于 15M
</br>
- 若 前一个区块使用 gas < 15M（低于目标） → 区块不拥堵：
Base Fee 会下降

- 若 前一个区块使用 gas > 15M（高于目标） → 区块拥堵：
Base Fee 会上升

- 若 前一个区块使用 gas = 15M → Base Fee 不变

* 举个例子

假设某个区块目标是 15M gas，但用了 22.5M：

拥堵程度 = (22.5M - 15M) / 15M = 0.5 → 拥堵 50%

Base Fee 增加最多 12.5%（因为这是调整上限）
# flashbot 架构
![image](https://github.com/user-attachments/assets/5c789960-3741-494b-87cb-975b50bc8c6a)
![image](https://github.com/user-attachments/assets/9c8a5cc0-bec4-4546-a784-ad35f4d0d4ff)

## searcher
是一个角色, 不是一个组件或者service
## builder
通过提供rpc service 来将交易直接发往特定的builder, 相当于交易完全私有, 不通过公开的mempool.
builder 执行一系列的算法来决定一个区块中应该包含哪些bundles和交易来最大化该区块的最大利润，


* 区块价值的评分（来源于mev-boost官方论坛）

  打包某一个区块前后矿工余额差值

* 区块价值来源

  base_fee + priority_fee + block.coinbase transfer + regular transfer

作为builder 只要有, 足够优秀的区块排序算法， 足够多的私有订单(保证盈利)， 足够多的P2P订单都可以做一个builder

* 订单来源

挑选公共pending pool,  searcher 和 private user 中 的交易进行选择性打包, 或者一些dex 项目方直接把rpc 换成某个builder 的, 

* 利润来源

builder 可以拿到gas 费的一部分, 例如priority fee(tip), 然后在区块中构建最后一笔交易直接打给builder 代码配置的地址
### 优势
用更优秀的排序交易算法使矿工能拿到更多的收益, 使用户的交易不透明避免MEV 攻击.

## relayer 
![image](https://github.com/user-attachments/assets/f6b6cdce-13e6-4c09-9990-cb0fd6b055c5)

builder 通过relayer 给validator 的交易只包含交易头, 防止validator 通过获取交易详情来又排序交易来套利. 
一些大的机构在运行relayer, eth 的relayer 有固定的一些合作伙伴.  

# Proposer-Builder Separation 这套架构的优势
## 为什么从传统 mempool 演进到 PBS？
传统 mempool 的问题：
所有交易都是公开的: 容易被 front-run、sandwich、MEV 抢跑

没有防作弊机制: 矿工可以选择性插入、删改、复制套利交易

用户体验差: 费用不可预测、交易顺序不公平

## 优势
* 安全性提升
Validator 不需要知道交易具体内容，防止作弊

Builder 和 Validator 相互不信任也能合作（通过中继）

* 效率提升
Builder 可以专注做极致利润排序（用自定义策略、MEV bot）

Validator 只需选“出价最高的 block”，不需关心排序算法

* 市场竞争 + 去中心化
多个 Builder 竞争，提高整体收益(但实际上目前市面上builder 的集中在几家上面, 其他小的builder 拿不到一些私有订单, 赚不到钱)

多个 Relay 转发，避免中心化控制

## 流程
1. Builder 构建一个包含高 MEV 的交易组合，价值 1 ETH

2. Builder 向 Relay 提交 block，表示愿意给 validator 分 0.9 ETH

3. Validator 收到多个 builder 区块，选择价值最高的那一个（0.9 ETH）

4. Block 被打包进链，validator 获得出块奖励，builder 获得套利差额
