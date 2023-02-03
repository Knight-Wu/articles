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


### TCP 
参考链接: 
 [https://en.wikipedia.org/wiki/Transmission_Control_Protoco](https://en.wikipedia.org/wiki/Transmission_Control_Protoco)
 [understand-tcp-ip-network-stack](https://cizixs.com/2017/07/27/understand-tcp-ip-network-stack/)
 https://www.jianshu.com/p/65605622234b

#### 建立连接

![enter image description here](https://drive.google.com/uc?id=1oroW4PjFfuKpe0BGhCTr1WlVMjvYbk3g) 

* 为什么需要三次握手

防止服务端接收到网络上滞留的请求连接包, 若没有第三次 ack 的过程, 服务端接收了此类包发送一个syn+ack, 并一直等待客户端发送数据, 导致浪费资源, 而且是死锁的情况, 若有第三次的ack, 则客户端收到syn+ack, 但是因为之前的包已经过时, 不会再次发送ack, 故服务器不会盲等.

#### 关闭连接

![enter image description here](https://drive.google.com/uc?id=1Uqsp8zQ1CHq2bwsThdbCDFo38K_23koR)

* 为什么需要四次挥手

因为TCP 是全双工的协议, 双方都可以发送和接收, 必须要一方发送结束信号, 另一方ack 之后才表示结束, 否则经过前两次挥手之后只是单向断开, 服务器还是能向客户端发送数据, 

* 为什么客户端关闭连接前要等待2MSL时间？为什么需要time-wait

1. 为了客户端最后一个ack 能确保到达服务器, 最后一个ack 有可能丢失, 则服务器会超时重传, 如果在2 MSL 时间内还没有超时重传, 证明已经接受到了最后一个ack, 若客户端不等待, 则无法再次发送最后一个ack, 服务器无法关闭连接. 所以需要time-wait, 来重发最后一个ack. 
2. 客户端发送完最后一个确认报文后，在这个2MSL时间中，就可以使本连接持续的时间内所产生的所有报文段都从网络中消失。这样新的连接中不会出现旧连接的请求报文。


 A connection can be ["half-open"](https://en.wikipedia.org/wiki/TCP_half-open "TCP half-open"), in which case one side has terminated its end, but the other has not. The side that has terminated can no longer send any data into the connection, but the other side can. The terminating side should continue reading the data until the other side terminates as well.
意思是从 FIN_WAIT_2 到 TIME_WAIT 状态是主动关闭方等待被动关闭方发送完所有消息







2. TCP 需要的资源

> Most implementations allocate an entry in a table that maps a session to a running operating system process. Because TCP packets do not include a session identifier, both endpoints identify the session using the client's address and port. Whenever a packet is received, the TCP implementation must perform a lookup on this table to find the destination process. Each entry in the table is known as a Transmission Control Block or TCB. It contains information about the endpoints (IP and port), status of the connection, running data about the packets that are being exchanged and buffers for sending and receiving data.

意思是 TCP packets 知道目标ip和端口, 但是不知道对应哪个进程, 需要在目标机器的一个map 表(TCP look up table)里面查询进程等相关信息.


> The number of sessions in the server side is limited only by memory and can grow as new connections arrive, but the client must allocate a random port before sending the first SYN to the server. This port remains allocated during the whole conversation, and effectively limits the number of outgoing connections from each of the client's IP addresses. If an application fails to properly close unrequired connections, a client can run out of resources and become unable to establish new TCP connections, even from other applications.

意思是作为TCP 连接的server 端, 只需要消耗内存, 因为可以多个连接都连到一个端口, 但是client 端却需要新建一个随机端口, 并在连接时一直持有, 若应用没有合理的释放资源, 会导致端口不够用.

#### TCP 报文首部
序号: 发送的报文中第一个字节的序号
确认号: 期望收到的下一个字节的序号, 若为N , 则表示到N-1 的所有序号的数据都已收到

> 控制位

紧急URG: 表示该报文需要马上接收到, 把紧急数据放到报文前部, 例如ctrl+c 的数据需要马上传输
确认ACK: 当ACK=1, 确认号才有效
推送PSH: 表示需要将数据立马上传到上层, 而不是等待缓存满了再发送
复位RST(reset): 置为1, 表示TCP 连接出现严重差错, 必须释放连接, 再重新建立. 
同步SYN: 表示连接请求或接收报文
终止FIN
窗口:接收方告诉发送方他目前接收的发送数据量
校验和
紧急指针: 紧急数据的长度


### TCP 为什么是可靠的传输

#### 实现无差错传输的解决方案
1. 停止-等待协议
每次发送一个分组就等待确认, 若确认收不到则超时重传, 但是信道利用率太低, 因为发送方的发送时间远小于两个分组的传输间隔(由发送时间, 往返时间, 发送确认时间组成)
2.  连续ARQ 协议
设定发送窗口和接收窗口, *
发送窗口: 
每接收到一个确认帧, 发送窗口才能前移一个, 若发送窗口所有帧都没有确认, 则需要等待确认才能前移,若超时没有确认, 则重发, 超时时间和报文的往返时间成正比, 
 接收窗口: 
每接收一个发送帧, 接收窗口前移, 当接收到一整个发送窗口时才发送一个ack,  在接收窗口之外的包一律丢弃


#### 网络的拥塞控制
* 慢开始和拥塞避免算法
拥塞窗口, 可能等于发送窗口, 但是也会受接收方建议的窗口大小的影响, 一开始先将拥塞窗口调的很小, 每收到一次完整的拥塞窗口确认之后, 则拥塞窗口翻倍. 呈指数增加,  当拥塞窗口到达一定阈值 a, 则采用拥塞避免算法, 线性增加. 当出现超时重传等情况时, 就有可能出现了拥塞, 则马上将阈值a 调为一半, 拥塞窗口调为1
思想为: 乘法减小, 加法增大, 拥塞的时候立马降低, 再缓慢恢复. 

* 快重传和快恢复
如果是发送超时重传的话, 使用的是慢开始和拥塞避免算法; 但是如果是接收方没有接收到中间的几个分组, 但是接收到了后面的几个新分组, 就立马向发送方发送重复确认(不需要等待自己发送的时候再捎带), 然后发送方立马重传没有接收的分组, 这叫快重传, 
而且因为接收方是接收到了后面的几个分组的, 拥塞并不严重, 所以发送方将拥塞窗口设定为阈值 a 的一半, 并采用拥塞避免算法, 线性增大. 

#### TCP_WAIT 过多是否异常
[2014-tcp-time-wait-state-linux](https://vincent.bernat.ch/en/blog/2014-tcp-time-wait-state-linux)
[https://blog.oldboyedu.com/tcp-wait/](https://blog.oldboyedu.com/tcp-wait/)
* time wait 过多的原因

因为客户端建立短连接, 新建后立马又断掉. 
正常的TCP客户端连接在关闭后，会进入一个TIME_WAIT的状态，持续的时间一般在2 MSL (一分钟)，对于连接数不高的场景，对系统也不会有什么影响，
但如果短时间内（例如1s内）进行大量的短连接，则可能出现这样一种情况：客户端所在的操作系统的socket端口和文件描述符被用尽，客户端无法再发起新的连接！

举例来说：
  假设每秒建立了1000个短连接（Web场景下是很常见的，例如每个请求都去访问memcached），假设TIME_WAIT的时间是1分钟，则1分钟内需要建立6W个短连接，由于TIME_WAIT时间是1分钟，这些短连接1分钟内都处于TIME_WAIT状态，都不会释放，而Linux默认的本地端口范围配置是：net.ipv4.ip_local_port_range = 32768, 因此这种情况下新的请求由于没有本地端口就不能建立了。

* time_wait 状态的作用

主动关闭方为客户端, 被动关闭方为服务器, 
1. 确保最后客户端的ack, 服务器可以收到, 如果服务器没收到则一定会在 2MSL 时间内重传最后一个Fin, 如果不等待, 则重传的 Fin 就收不到, 服务器就会处于last ack 状态一直等待, 浪费服务器资源, 导致服务器端口耗尽. 

2. 这个可以不说, 等待2 MSL 的时间后, 这个链接的消息在网络上都消失, 避免导致后续新建的连接(如果源ip, 源端口, 目的ip, 目的端口,协议都和A 连接相同的)收到, 导致数据错乱

* time_wait 的危害

1. 占用 Connection table slot

简而言之, 会导致client 无法同一时间发起更多的连接, 因为time_wait 释放需要一分钟, 每秒新建 500 个conn 就会占满, 此时会占用 Connection table 这样一个四元组 (source address, source port, destination address, destination port), 而这个四元组同一时刻只能有一个.

2. 内存的消耗

With many connections to handle, leaving a socket open for one additional minute may cost your server some memory. For example, if you want to handle about 10,000 new connections per second, you will have about 600,000 sockets in the  `TIME-WAIT`  state. How much memory does it represent? Not that much!

First, from the application point of view, a  `TIME-WAIT`  socket does not consume any memory: the socket has been closed.
实际上 a  `TIME-WAIT`  socket并不消耗应用内存. 所占系统内核的内存也很少, 最多不过几十兆.

3. cpu消耗也很少

> 总之 TCP_WAIT 在几百几千是挺正常的, 几万才算多, 最大值取决于当前节点配置的客户端端口数量,  如果应用层面没出现异常, 可以忽略.

* 如何解决time-wait 过多

短连接过多或突然客户端关闭大量链接, 在客户端出现大量time wait. 
https://cloud.tencent.com/developer/article/1004354

* close_wait过多原因

close_wait 出现在服务端, 被动关闭方, 简而言之就是没有发送 Fin 包, FIN包的底层实现其实就是调用socket的close方法，这里的问题出在正常执行close方法。

close_wait 按照正常操作的话应该很短暂的一个状态，接收到客户端的fin包并且回复客户端ack之后，会继续发送FIN包告知客户端关闭关闭连接，之后迁移到Last_ACK状态。但是close_wait过多只能说明没有迁移到Last_ACK，也就是服务端是否发送FIN包，只有发送FIN包才会发生迁移，所以问题定位在是否发送FIN包。

## big-endian vs little-endian

大端: 就是把低字节存在高位, 方便阅读和debug, 方便人从上到下就可以顺序读取整个字节
小端: 就是把低字节存在低位, 方便先读低字节, 例如判断奇数和偶数. 