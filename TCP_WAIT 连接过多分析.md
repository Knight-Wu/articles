### 背景

[2014-tcp-time-wait-state-linux](https://vincent.bernat.ch/en/blog/2014-tcp-time-wait-state-linux)
### 需要提前掌握的知识


* time_wait 状态的作用
1. 确保被动关闭方B 能知道主动关闭方 A 已经完全关闭了, 否则会一直处于LAST_ACK 的状态, 认为旧有的连接没有完全关闭, 进而有可能导致新建的连接也失败.
> Without the `TIME-WAIT` state, a connection could be reopened while the remote end still thinks the previous connection is valid. When it receives a _SYN_ segment (and the sequence number matches), it will answer with a _RST_ as it is not expecting such a segment. The new connection will be aborted with an error

time_wait 会等待60秒后关闭.
> [RFC 793](https://tools.ietf.org/html/rfc793 "RFC 793: Transmission Control Protocol")  requires the  `TIME-WAIT`  state to last twice the time of the  MSL. On Linux, this duration is  **not**  tunable and is defined in  `include/net/tcp.h`  as one minute:
define TCP_TIMEWAIT_LEN (60*HZ) /* how long to wait to destroy TIME-WAIT
  state, about 60 seconds. There have been  [propositions to turn this into a tunable value](http://web.archive.org/web/2014/http://comments.gmane.org/gmane.linux.network/244411 "[RFC PATCH net-next] tcp: introduce tcp_tw_interval to specifiy the time of TIME-WAIT")  but it has been refused on the ground the  `TIME-WAIT`  state is a good thing.

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTc3ODUzNTExMywxMTk0OTAxMTg3LDE4OD
k3OTA3NTgsLTE1MTQ3NDQzOTFdfQ==
-->