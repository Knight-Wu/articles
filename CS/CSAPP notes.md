#### [locality of reference](https://en.wikipedia.org/wiki/Locality_of_reference "Locality of reference").
局部性原理: 
分为时间, 空间和算法三个局部性
* 时间局部性
* 空间局部性
未来需要使用的数据, 有可能在最近历史数据的附近. 
应用: loop_exchange , compiler 将loop 改写成读取符合数据在 memory 存储的形式, 例如如果一行的数据在memory 顺序存放, 则, 将最近需要的数据都从memory  一次性拿到cpu cache 缓存, 提高数据的提取速率, 
https://en.wikipedia.org/wiki/Loop_interchange 
* 算法局部性



> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMjk3MTQ0NTQ0XX0=
-->