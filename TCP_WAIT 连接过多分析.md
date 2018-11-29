### 背景

[2014-tcp-time-wait-state-linux](https://vincent.bernat.ch/en/blog/2014-tcp-time-wait-state-linux)
### 需要提前掌握的知识
![enter image description here](https://drive.google.com/uc?id=1Uqsp8zQ1CHq2bwsThdbCDFo38K_23koR)

[understand-tcp-ip-network-stack](https://cizixs.com/2017/07/27/understand-tcp-ip-network-stack/)
#### TCP 回顾
[https://en.wikipedia.org/wiki/Transmission_Control_Protoco](https://en.wikipedia.org/wiki/Transmission_Control_Protoco)
1. 关闭过程
> 重点语句

> A connection can be ["half-open"](https://en.wikipedia.org/wiki/TCP_half-open "TCP half-open"), in which case one side has terminated its end, but the other has not. The side that has terminated can no longer send any data into the connection, but the other side can. The terminating side should continue reading the data until the other side terminates as well.
意思是从 FIN_WAIT_2 到 TIME_WAIT 状态是主动关闭方等待被动关闭方发送完所有消息



2. TCP 需要的资源

> Most implementations allocate an entry in a table that maps a session to a running operating system process. Because TCP packets do not include a session identifier, both endpoints identify the session using the client's address and port. Whenever a packet is received, the TCP implementation must perform a lookup on this table to find the destination process. Each entry in the table is known as a Transmission Control Block or TCB. It contains information about the endpoints (IP and port), status of the connection, running data about the packets that are being exchanged and buffers for sending and receiving data.

意思是 TCP packets 知道目标ip和端口, 但是不知道对应哪个进程, 需要在目标机器的一个map 表(TCP look up table)里面查询进程等相关信息.


> The number of sessions in the server side is limited only by memory and can grow as new connections arrive, but the client must allocate a random port before sending the first SYN to the server. This port remains allocated during the whole conversation, and effectively limits the number of outgoing connections from each of the client's IP addresses. If an application fails to properly close unrequired connections, a client can run out of resources and become unable to establish new TCP connections, even from other applications.

意思是作为TCP 连接的server 端, 只需要消耗内存, 因为可以多个连接都连到一个端口, 但是client 端却需要新建一个随机端口, 并在连接时一直持有, 若应用没有合理的释放资源, 会导致端口不够用.
> Written with [StackEdit](https://stackedit.io/).

#### time_wait
[2014-tcp-time-wait-state-linux](https://vincent.bernat.ch/en/blog/2014-tcp-time-wait-state-linux)

* time_wait 状态的作用
1. 确保被动关闭方B 能知道主动关闭方 A 已经完全关闭了, 否则会一直处于LAST_ACK 的状态, 认为旧有的连接没有完全关闭, 进而有可能导致新建的连接也失败.
> Without the `TIME-WAIT` state, a connection could be reopened while the remote end still thinks the previous connection is valid. When it receives a _SYN_ segment (and the sequence number matches), it will answer with a _RST_ as it is not expecting such a segment. The new connection will be aborted with an error

time_wait 会等待60秒后关闭.
> [RFC 793](https://tools.ietf.org/html/rfc793 "RFC 793: Transmission Control Protocol")  requires the  `TIME-WAIT`  state to last twice the time of the  MSL. On Linux, this duration is  **not**  tunable and is defined in  `include/net/tcp.h`  as one minute:
define TCP_TIMEWAIT_LEN (60*HZ) /* how long to wait to destroy TIME-WAIT
  state, about 60 seconds. There have been  [propositions to turn this into a tunable value](http://web.archive.org/web/2014/http://comments.gmane.org/gmane.linux.network/244411 "[RFC PATCH net-next] tcp: introduce tcp_tw_interval to specifiy the time of TIME-WAIT")  but it has been refused on the ground the  `TIME-WAIT`  state is a good thing.

* time_wait 的危害

1. 占用 Connection table slot

A connection in the  `TIME-WAIT`  state is kept for one minute in the connection table. This means, another connection with the same  _quadruplet_  (source address, source port, destination address, destination port) cannot exist.

For a web server, the destination address and the destination port are likely to be constant. If your web server is behind a L7 load-balancer, the source address will also be constant. On Linux, the client port is by default allocated in a port range of about 30,000 ports (this can be changed by tuning  `net.ipv4.ip_local_port_range`). This means that only 30,000 connections can be established between the web server and the load-balancer every minute, so about  **500 connections per second**.

If the  `TIME-WAIT`  sockets are on the client side, such a situation is easy to detect. The call to  `connect()`  will return  `EADDRNOTAVAIL`  and the application will log some error message about that. On the server side, this is more complex as there is no log and no counter to rely on. 
简而言之, 会导致client 无法同一时间发起更多的连接, 因为time_wait 释放需要一分钟, 此时会占用 Connection table 这样一个四元组 (source address, source port, destination address, destination port), 而这个四元组同一时刻只能有一个.

2. 内存的消耗

With many connections to handle, leaving a socket open for one additional minute may cost your server some memory. For example, if you want to handle about 10,000 new connections per second, you will have about 600,000 sockets in the  `TIME-WAIT`  state. How much memory does it represent? Not that much!

First, from the application point of view, a  `TIME-WAIT`  socket does not consume any memory: the socket has been closed.
实际上 a  `TIME-WAIT`  socket并不消耗应用内存. 所占系统内核的内存也很少, 最多不过几十兆.
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTM5MjcwOTkwMCwtNjIyOTQ5NTk1LC0xMD
I4NDk1ODE4LC0xMDE2MDUzMjkwLDIwMjczODEzODMsMTc4OTI2
MDE3MiwtMTcwNjIwNzk1Miw1MDg1OTIzNjIsLTQ1MDc3NzU2MS
wtNzc4NTM1MTEzLDExOTQ5MDExODcsMTg4OTc5MDc1OCwtMTUx
NDc0NDM5MV19
-->