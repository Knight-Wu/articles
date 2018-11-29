### 背景

### 需要提前掌握的知识


* tcp_wait 状态的作用
1. 确保被动关闭方B 能知道主动关闭方 A 已经完全关闭了, 否则会一直处于LAST_ACK 的状态, 认为旧有的连接没有完全关闭, 进而有可能导致新建的连接也失败.
> Without the `TIME-WAIT` state, a connection could be reopened while the remote end still thinks the previous connection is valid. When it receives a _SYN_ segment (and the sequence number matches), it will answer with a _RST_ as it is not expecting such a segment. The new connection will be aborted with an error

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTE5NDkwMTE4NywxODg5NzkwNzU4LC0xNT
E0NzQ0MzkxXX0=
-->