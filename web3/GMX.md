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
