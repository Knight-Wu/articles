## RDBMS
### mysql 主备
* master-slave
缺陷指的是: slave 同步binlog 落后, 如果master crash , 那么中间的binlog 就会全部丢失, 而且master 是存在单节点 down 风险

* master-master
双主多备, 并且两台master 互为主备, 能通过virtual ip 提供对外服务, 降低一个master down 的风险, 并且降低主从同步对单个master 的压力, 但是缺点就是slave 需要等待masterB 同步完成之后才能同步, 慢了一点. 

* master high available
需要用 MHA manager(MHAM) 自动failover, MHAM 定期检测master 的状态, 若出现问题, 则找到具有最新进度的slave (如何找? ), 并将master 和该slave 的差异数据补上, 进一步最小化数据差异

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTExNjk0NDkzNTYsLTIxMTk2NzA0MzEsOT
Y4MTcyNjgxLC04NTE1OTEwODgsMTY2OTg0MTQyMCwtMTYzNTgy
MTc3NV19
-->