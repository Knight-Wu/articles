# 永续合约
## GMX
* 价格由chainlink 决定, 可以设置limit order. 如何实现limit order ?
* 用户杠杆的钱通过LP 池子借, 需要付借款利息, 对手方是LP, 这个LP 池子由多个币种组成, 每个币种按照价格乘数量计算在池子里面的比重, 这样流动性深度就比较深. 

* LP 存入和提现都收手续费, 通过调整手续费的高低来平衡池子里面各个token 的比重, 同样swap 的时候也调整手续费来影响池子里面的权重. 
### contract for difference
CFD(差价合同), 合规要求保证金由机构补足, 
![image](https://github.com/user-attachments/assets/46266c46-63ca-4fbf-bea0-0f62ec011113)



### 混币池的设计
![image](https://github.com/user-attachments/assets/d093cbdb-5195-481a-aeca-e012aa1e38aa)

![image](https://github.com/user-attachments/assets/b19ecb7b-7793-479d-8e96-4e93948b16a9)

### LP 提供流动性
![image](https://github.com/user-attachments/assets/26374fb3-8106-4b16-8bce-03e4af33451e)

### short, long
![image](https://github.com/user-attachments/assets/863f0d71-cf00-4174-a3fd-35825134c25c)

### 开仓限制
![image](https://github.com/user-attachments/assets/9f8ea0b1-f4dc-48e6-9b27-405497989372)


### token 的价格如何设置
总结说来就是chainlink 的价格浮动百分之2.5%, 项目方可以在这个2.5 % 的范围内设置一个价格. 
![image](https://github.com/user-attachments/assets/ace69d00-b785-4846-b8d1-4623f79ffd8b)

### 如何开市价单和限价单
![image](https://github.com/user-attachments/assets/3c1824ce-bf52-4696-9407-f5d5eaadde7a)

一 限价单交易代码: OrderBook
二 市价单交易 PositionRouter

#### 限价单
![image](https://github.com/user-attachments/assets/1dbe13af-aa71-4eac-8313-87cc11c1d5cd)
* 创建限价单
1. 做基础校验,
2. 订单保存在链上

* 增加仓位
executeIncreaseOrder 是具体执行加仓的逻辑
1. 判断当前账户下，指定索引的订单是否存在，若不存在，则直接报错
2. 验证当前的市场价格是否满足 价格触发条件，（如果是多头，用最大价格，空头用最小价格）
3. 删除对应的限价单，防止重复执行
4. 将购买到的token转移到Vault合约
5. 如果加仓代币和抵押代币不一致，需要调用内部转换逻辑进行转换，然后计算成抵押代币的数量，转移到Vault
6. 执行的是合约增加仓位的方法Vault.increasePosition，

#### 市价订单
![image](https://github.com/user-attachments/assets/5e0320b3-89be-4382-9ae3-e092b340bcb1)
1. 创建市价订单保存在链上, createIncreasePosition
2. 若满足条件, 交易机器人执行市价单的逻辑, executeIncreasePosition

#### Vault
* 增加头寸方法increasePosition
即执行限价单和市价单之后最终的执行方法
### 清算
![image](https://github.com/user-attachments/assets/8d38bc72-a34d-42a7-8eb4-b535c15fc4e5)
