#### [locality of reference](https://en.wikipedia.org/wiki/Locality_of_reference "Locality of reference").
局部性原理: 
分为时间, 空间和算法三个局部性
* 时间局部性
* 空间局部性
未来需要使用的数据, 有可能在最近历史数据的附近. 
应用:
https://stackoverflow.com/questions/11276291/why-cant-or-doesnt-the-compiler-optimize-a-predictable-addition-loop-into-a 
[loop_exchange](https://en.wikipedia.org/wiki/Loop_interchange)
 compiler 将多层 loop 改写成读取符合数据在 memory 存储的形式, 例如如果一行的数据在memory 顺序按照一列一列的存放, 则一次性读取同一行几列的数据放到cpu cache 中,  提高缓存命中率, 

* 算法局部性

#### processor [branch predictor](https://en.wikipedia.org/wiki/Branch_predictor)
简而言之: cpu 不知道下个指令具体怎么执行, 除非知道上一个指令执行的结果, 所以有两种方法: 等待上一个指令执行的结果, 或者根据历史预测, 通常情况下预测的命中性达到 90 %, 但是当历史结果是非常随机不可预测的情况下, branch predictor 就可能效率比较低, 预测错了之后还需要回溯再次执行. 
 
* 应用
https://stackoverflow.com/questions/11227809/why-is-it-faster-to-process-a-sorted-array-than-an-unsorted-array/11227902#11227902
> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTU4NDg1NjkxMl19
-->