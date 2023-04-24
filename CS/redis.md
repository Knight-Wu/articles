
### redis pipeline 和 transaction 的区别
pipeline 主要一是为了减少客户端和服务端的交互, 二是下一个命令不用等待上一个命令执行的结果, 多个命令一起flush 到服务端, 然后执行一条, 得到一条命令的结果, 缓存, 然后再将多个结果返回到客户端. 

而transaction 则是会缓存多条命令, 一起执行, 再一起返回结果, 但是一个命令成功, 有些的失败了, 成功的不能回滚, 并且中间不能插其他的命令, 而pipeline 则可以插其他命令.

## redis 如何做 unique userid(uv) 的统计, 需要去重
1. bitset
<img width="794" alt="image" src="https://user-images.githubusercontent.com/20329409/233603506-1e0ae876-d2c1-4caf-9f75-3a040eb062a1.png">

存储空间取决于 offset 的最大值, 例如存 int, 0 到 2 的 32 次方 -1 , 需要 2^32 -1 bytes =  512MB, 如果多个 bitset 合并则只需要按位与即可. 
时间复杂度只有第一次申请空间的时候较久, 之后都是 O(1), 如果要查 bitset 里面实际有多少个 bit 则使用 BITCOUNT

* 如何实现 bitset 呢

实际上是一个 bit 数组, 例如 数组长度是 2 的 32 次方 - 1, 那么空间就是  2 的 32 次方 - 1 个 bit, 就是 512 MB, 那么如果设置 offset = 10(十进制) 为 1, 那么就是在数组的第十个位置置 1. 

借助 bitset 和 redission lib 可以实现 bloomfilter .
新增元素的过程中可以统计 unique element 的个数, 只要某个 bit 位之前是 0, 被设置成 1 , 那么就是一个不同的 key, 虽然也会因为 fpp 造成独立元素的个数偏少.
2. hyperloglog

牺牲了一点准确度, 但是空间却非常少.
用途用于计算城市的道路拥挤程度, 就是每个街道的不重复的车牌数量, 有一辆车进入的时候就 pfAdd blockNum licenseNum, 离开的时候新的 block 
pfadd, 旧的 block 直接在计数器上减一, 因为肯定知道这辆车从哪个街道过来的.
