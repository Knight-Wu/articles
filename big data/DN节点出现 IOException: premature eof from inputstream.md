### 问题背景
生产cloudera 集群dn 节点频繁出现IOException: premature eof from inputstream, 怀疑是client 因为网络问题突然断掉连接, 数据写入中断导致, 进而去找CM 上面的监控网络的图表, 发现TCP_WAIT 比较多, 进而产生两个问题: 

1. IOException: premature eof from inputstream 出现的根因
2. TCP_WAIT 过多是否异常


#### TCP/IP协议
　　既然是网络编程，涉及几个系统之间的交互，那么首先要考虑的是如何准确的定位到网络上的一台或几台主机，另一个是如何进行可靠高效的数据传输。这里就要使用到TCP/IP协议。

　　TCP/IP协议（传输控制协议）由网络层的IP协议和传输层的TCP协议组成。IP层负责网络主机的定位，数据传输的路由，由IP地址可以唯一的确定Internet上的一台主机。TCP层负责面向应用的可靠的或非可靠的数据传输机制，这是网络编程的主要对象。

#### TCP与UDP
　　TCP是一种面向连接的保证可靠传输的协议。通过TCP协议传输，得到的是一个顺序的无差错的数据流。发送方和接收方的成对的两个socket之间必须建立连接，以便在TCP协议的基础上进行通信，当一个socket（通常都是server socket）等待建立连接时，另一个socket可以要求进行连接，一旦这两个socket连接起来，它们就可以进行双向数据传输，双方都可以进行发送或接收操作。

　　UDP是一种面向无连接的协议，每个数据报都是一个独立的信息，包括完整的源地址或目的地址，它在网络上以任何可能的路径传往目的地，因此能否到达目的地，到达目的地的时间以及内容的正确性都是不能被保证的。

TCP与UDP区别：

TCP特点：

　　1、TCP是面向连接的协议，通过三次握手建立连接，通讯完成时要拆除连接，由于TCP是面向连接协议，所以只能用于点对点的通讯。而且建立连接也需要消耗时间和开销。

　　2、TCP传输数据无大小限制，进行大数据传输。

　　3、TCP是一个可靠的协议，它能保证接收方能够完整正确地接收到发送方发送的全部数据, 而且是顺序的.

UDP特点：

　　1、UDP是面向无连接的通讯协议，UDP数据包括目的端口号和源端口号信息，由于通讯不需要连接，所以可以实现广播发送。

　　2、UDP传输数据时有大小限制，每个被传输的数据报必须限定在64KB之内。

　　3、UDP是一个不可靠的协议，发送方所发送的数据报并不一定以相同的次序到达接收方(不保证顺序)。

TCP与UDP应用：

　　1、TCP在网络通信上有极强的生命力，例如远程连接（Telnet）和文件传输（FTP）都需要不定长度的数据被可靠地传输。但是可靠的传输是要付出代价的，对数据内容正确性的检验必然占用计算机的处理时间和网络的带宽，因此TCP传输的效率不如UDP高。

　　2，UDP操作简单，而且仅需要较少的监护，因此通常用于局域网高可靠性的分散系统中client/server应用程序。例如视频会议系统，并不要求音频视频数据绝对的正确，只要保证连贯性就可以了，这种情况下显然使用UDP会更合理一些。


#### TCP 回顾
参考链接: 
 [https://en.wikipedia.org/wiki/Transmission_Control_Protoco](https://en.wikipedia.org/wiki/Transmission_Control_Protoco)
 [understand-tcp-ip-network-stack](https://cizixs.com/2017/07/27/understand-tcp-ip-network-stack/)



1. 关闭连接

![enter image description here](https://drive.google.com/uc?id=1Uqsp8zQ1CHq2bwsThdbCDFo38K_23koR)

2. 建立连接

![enter image description here](https://drive.google.com/uc?id=1oroW4PjFfuKpe0BGhCTr1WlVMjvYbk3g) 


> 重点语句

> A connection can be ["half-open"](https://en.wikipedia.org/wiki/TCP_half-open "TCP half-open"), in which case one side has terminated its end, but the other has not. The side that has terminated can no longer send any data into the connection, but the other side can. The terminating side should continue reading the data until the other side terminates as well.
意思是从 FIN_WAIT_2 到 TIME_WAIT 状态是主动关闭方等待被动关闭方发送完所有消息



2. TCP 需要的资源

> Most implementations allocate an entry in a table that maps a session to a running operating system process. Because TCP packets do not include a session identifier, both endpoints identify the session using the client's address and port. Whenever a packet is received, the TCP implementation must perform a lookup on this table to find the destination process. Each entry in the table is known as a Transmission Control Block or TCB. It contains information about the endpoints (IP and port), status of the connection, running data about the packets that are being exchanged and buffers for sending and receiving data.

意思是 TCP packets 知道目标ip和端口, 但是不知道对应哪个进程, 需要在目标机器的一个map 表(TCP look up table)里面查询进程等相关信息.


> The number of sessions in the server side is limited only by memory and can grow as new connections arrive, but the client must allocate a random port before sending the first SYN to the server. This port remains allocated during the whole conversation, and effectively limits the number of outgoing connections from each of the client's IP addresses. If an application fails to properly close unrequired connections, a client can run out of resources and become unable to establish new TCP connections, even from other applications.

意思是作为TCP 连接的server 端, 只需要消耗内存, 因为可以多个连接都连到一个端口, 但是client 端却需要新建一个随机端口, 并在连接时一直持有, 若应用没有合理的释放资源, 会导致端口不够用.
> Written with [StackEdit](https://stackedit.io/).

#### 解答TCP_WAIT 过多是否异常
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
eyJoaXN0b3J5IjpbLTEyMjc4NjU0MjddfQ==
-->