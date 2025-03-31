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
通过提供rpc service 来将交易直接发往特定的builder, 相当于交易完全私有, 不通过公开的mempool, 
## relayer 
![image](https://github.com/user-attachments/assets/f6b6cdce-13e6-4c09-9990-cb0fd6b055c5)

builder 通过relayer 给validator 的交易只包含交易头, 防止validator 通过获取交易详情来又排序交易来套利. 
一些大的机构在运行relayer, eth 的relayer 有固定的一些合作伙伴.  
