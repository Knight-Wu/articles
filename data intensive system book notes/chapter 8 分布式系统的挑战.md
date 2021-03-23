* 需要一个集中式的单挑递增, 与时间无关的计数器用于分发事务 ID 等, 因为多节点中, 时间和网络是不可信的, 或者说是很难实际同步的. 
* 该事务或者事件 ID 用于哪里呢: 用于多个客户端写存储的时候判断顺序性, 或者判断租约有效性, 例如当客户端 A 因为 GC 暂停等原因导致进程暂停, 而租约实际已


> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTMzNTA1MzMwNF19
-->