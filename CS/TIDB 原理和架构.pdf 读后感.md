## RDBMS
### mysql 主备
* master-slave
缺陷指的是: slave 同步binlog 落后, 如果master crash , 那么中间的binlog 就会全部丢失, 而且master 是存在单节点 down 风险

* master-master
双主多备, 并且两台master 互为主备, 能通过vir t

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE1MDI3ODg2ODYsMTY2OTg0MTQyMCwtMT
YzNTgyMTc3NV19
-->