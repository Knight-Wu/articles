## RDBMS
### mysql 主备
* master-slave
缺陷指的是: slave 同步binlog 落后, 如果master crash , 那么中间的binlog 就会全部丢失, 而且master 是存在单节点 down 风险

* Master-Master replication manager for MySQL(MMM)
双主多备, 并且两台master 互为主备, 能通过virtual ip 提供对外服务, 降低一个master down 的风险, 并且降低主从同步对单个master 的压力, 满足了**高可用, 但是不满足数据一致性**, 因为masterB 在masterA 宕机, 切换读请求到 masterB 时如果数据并没有完成和 masterA 的同步, 那么就会造成数据不一致, 
(例如此时数据落后于与A 的同步, 后续的更新请求可能就会不满足where 条件)

* master high available(MHA)
需要用 MHA manager(MHAM) 自动failover, MHAM 定期检测master 的状态, 若出现问题, 则找到具有最新进度的slave (如何找? ), 并将master 和该slave 的差异数据补上, 进一步最小化数据差异. 

* mysql group replication (5.7.17)


> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMjAzMTc5Mzc0LC0xMzY1OTQ3MTE0LDIzMz
AwMjY4NCwxOTk5Nzg1MzA0LC0yMTE5NjcwNDMxLDk2ODE3MjY4
MSwtODUxNTkxMDg4LDE2Njk4NDE0MjAsLTE2MzU4MjE3NzVdfQ
==
-->