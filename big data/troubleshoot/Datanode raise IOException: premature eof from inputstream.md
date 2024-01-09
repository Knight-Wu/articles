### 问题背景
生产cloudera 集群dn 节点频繁出现IOException: premature eof from inputstream, 怀疑是client 因为网络问题突然断掉连接, 数据写入中断导致, 进而去找CM 上面的监控网络的图表, 发现TCP_WAIT 比较多, 进而产生两个问题: 

1. IOException: premature eof from inputstream 出现的根因
2. TCP_WAIT 过多是否异常




#### TCP_WAIT 过多是否异常
[2014-tcp-time-wait-state-linux](https://vincent.bernat.ch/en/blog/2014-tcp-time-wait-state-linux)
[https://blog.oldboyedu.com/tcp-wait/](https://blog.oldboyedu.com/tcp-wait/)
* time_wait 状态的作用

1. 确保前一个连接A 发的消息因为网络阻塞,  A连接关闭后, 可能被后续新建的连接(如果源ip, 源端口, 目的ip, 目的端口,协议都和A 连接相同的)收到, 导致数据错乱

> The most known one is to **prevent delayed segments** from one connection being accepted by a later connection relying on the same quadruplet (source address, source port, destination address, destination port). The sequence number also needs to be in a certain range to be accepted. This narrows a bit the problem but it still exists, especially on fast connections with large receive windows. [RFC 1337](https://tools.ietf.org/html/rfc1337 "RFC 1337: TIME-WAIT Assassination Hazards in TCP") explains in details what happens when the `TIME-WAIT`state is deficient.

2. 确保被动关闭方B 能知道主动关闭方 A 已经完全关闭了, 否则会一直处于LAST_ACK 的状态, 认为旧有的连接没有完全关闭, 占用一个socket 五元组, 并且有可能导致新建的连接失败.
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

3. cpu消耗也很少

* 其他解决time_wait的办法见上面链接的原文. 

> 总之 TCP_WAIT 在几百几千是挺正常的, 几万才算多, 最大值取决于当前节点配置的客户端端口数量,  如果应用层面没出现异常, 可以忽略.


#### IOException: premature eof from inputstream 出现的根因
根据这个JIRA [HDFS-9572](https://issues.apache.org/jira/browse/HDFS-9572), 
> Monitoring tools may choose to check liveness of the DataNode's data transfer port by connecting to it. The monitoring tool will close the connection immediately after establishment without sending any data. When this happens, the DataNode encounters an unexpected EOF and logs a full stack trace. This creates unneeded noise in the logs.


 新版已经合并了这个patch,  [hadoop-2.8 DataXceiver.java](https://github.com/apache/hadoop/blob/branch-2.8/hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/server/datanode/DataXceiver.java), line 272
<!--stackedit_data:
eyJoaXN0b3J5IjpbNTE2MDE5OTU5LC0xMjI3ODY1NDI3XX0=
-->
