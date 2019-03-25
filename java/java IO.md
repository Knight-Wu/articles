
---

#### java IO的历史
从一开始的Socket BIO, 同步阻塞, 等待连接和读写的时候是阻塞的, 只能忙等,并且一个连接必须开多一个线程去接受否则会因为一个客户端慢导致其他客户端受影响; 若开多个线程会导致线程切换开销大, 无法应对海量连接; 到1.4的 NIO, 同步非阻塞, 连接和读写请求非阻塞, 单线程采用selector去接受请求, 再根据事件的不同用不同的线程去处理, 但是读写,创建连接仍然是同步的, 需要等待读写,连接的完成才能进行业务处理; 再到 1.7的NIO2(也可以叫做 AIO ,异步非阻塞IO), 则是读写以及连接完了通知下一步处理, 采用回调函数进行业务处理, 完全异步的形式, 适用于读写过程长的任务, 不用等待读写完成.




#### 服务器单线程处理
> 可能一个客户端慢导致其他客户端受影响
    
#### 服务器多线程处理
*问题* 
 1.1 可以开启cpu核心数个线程进行并发处理, 但是如何处理后续海量连接等问题...

#### nio处理, 主线程根据selector获取到不同的事件(包括连接, 写和读)
主线程轮询selector, 获取到accept 事件后, 表明有新连接进来, 并把这个新连接注册一个读事件到selector 上, 当这个连接可读的时候, IO 线程就会处理读事件, 完成连接的读.  在处理完IO后，业务一般还会有自己的业务逻辑，有的还会有其他的阻塞IO，如DB操作，RPC等。只要有阻塞，就需要新起一个线程, 否则当前线程. 
但是仍然会面对 1.1的问题
```

interface ChannelHandler{
      void channelReadable(Channel channel);
      void channelWritable(Channel channel);
   }
   class Channel{
     Socket socket;
     Event event;//读，写或者连接
   }

   //IO线程主循环:
   class IoThread extends Thread{
   public void run(){
   Channel channel;
   while(channel=Selector.select()){//选择就绪的事件和对应的连接
      if(channel.event==accept){
         registerNewChannelHandler(channel);//如果是新连接，则注册一个新的读写处理器
      }
      if(channel.event==write){
         getChannelHandler(channel).channelWritable(channel);//如果可以写，则执行写事件
      }
      if(channel.event==read){
          getChannelHandler(channel).channelReadable(channel);//如果可以读，则执行读事件
      }
    }
   }
   Map<Channel，ChannelHandler> handlerMap;//所有channel的对应事件处理器
  }

```

* buffer
jdk 文档如下: 
https://docs.oracle.com/javase/7/docs/api/java/nio/channels/SocketChannel.html#read(java.nio.ByteBuffer[])

>Reads a sequence of bytes from this channel into the given buffer.
An attempt is made to read up to  _r_  bytes from the channel, where  _r_  is the number of bytes remaining in the buffer, that is,  dst.remaining(), at the moment this method is invoked.
// 故需要在每次读满buffer 之后, 进行flip(buffer 准备写), 或者
clear, 保证还有remaning, 因为remaning = limit- position

> A read operation might not fill the buffer, and in fact it might not read any bytes at all. Whether or not it does so depends upon the nature and state of the channel. A socket channel in non-blocking mode, for example, cannot read any more bytes than are immediately available from the socket's input buffer; similarly, a file channel cannot read any more bytes than remain in the file. It is guaranteed, however, that if a channel is in blocking mode and there is at least one byte remaining in the buffer then this method will block until at least one byte is read.
// read 方法是不是阻塞的取决于channel 是不是blocking, 

示例如图: 
![enter image description here](https://drive.google.com/uc?id=1vpf2cAFewS2ODiCrEKZkJporZcRyId8O)

![enter image description here](https://drive.google.com/uc?id=1GExlB0WE2cGSTpjmZNEdY_oygxB0KD95)

缓冲区, byteBuffer看看源码很简单, 有position 表示读写到哪了, mark 只是个标记位, 用来回溯, limit 表示可以读写的边界, capacity 表示字节数组的大小, 来控制字节数组读写的位置, 读之前需要flip() 回到数组头部开始读.

 * DirectByteBuffer, HeapByteBuffer
  DirectByteBuffer 字节数组直接存储在native memory, 避免了native memory和java heap 的来回拷贝, 但是分配和回收native memory的速度会比heap memory要慢;
        在网络读写和文件读写的时候, 基于buffer 进行读写, 需要保证buffer 的地址在读写时不能改变(调用操作系统函数的时候, 传入buffer的起始地址和size), 而GC的时候很有可能改变buffer的地址, 所以需要将buffer 先拷贝到堆外内存, 就是native memory, 而HeapByteBuffer的数组一开始是在java heap的, 故需要先拷贝到native memory, 因此效率会比 DirectByteBuffer 要慢.
        
1. 如何GC的
            
 GC压力更小。虽然GC仍然管理着DirectBuffer的回收，但它是使用PhantomReference来达到的，在平常的Young GC或者mark and compact的时候却不会在内存里搬动。如果IO的数量比较大，比如在网络发送很大的文件，那么GC的压力下降就会很明显。但是具体GC的细节和发生条件, 和时间还不清楚, 可以参见这篇帖子 [PhantomReference & Cleaner](https://zhuanlan.zhihu.com/p/29454205)
            
 >    // Doubly-linked list of live cleaners, which prevents the cleaners themselves from being GC'd before their referents ??
        
     底层通过write、read、pwrite，pread函数进行系统调用时，需要传入buffer的起始地址和buffer count作为参数。具体参见：write(2): to file descriptor，read(2): read from file descriptor，pread(2) - Linux man page，pwrite(2) - Linux man page。如果使用java heap的话，我们知道jvm中buffer往往以byte[] 的形式存在，这是一个特殊的对象，由于java heap GC的存在，这里对象在堆中的位置往往会发生移动，移动后我们传入系统函数的地址参数就不是真正的buffer地址了，这样的话无论读写都会发生出错。而C Heap仅仅受Full GC的影响，相对来说地址稳定 
        
2. 传统的文件IO的流程
 * MappedByteBuffer 
   可以作为利用内存来映射region of file, 可以快速进行大文件的读写, 详见 jdk官方文档.

* channel
> A Java NIO FileChannel is a channel that is connected to a file. Using a file channel you can read data from a file, and write data to a file. The Java NIO FileChannel class is NIO's an alternative to reading files with the standard Java IO API. A FileChannel cannot be set into non-blocking mode. It always runs in blocking mode.

* selector
用于单线程去接受网络事件, 包括读写,连接,接收; 避免了多线程接收连接导致的资源消耗高, 线程上下文切换慢, 也不能解决海量连接的问题.

* 同步与异步, 阻塞与非阻塞
    同步指的是主线程需要轮询事件是否完成, 异步则是事件完成后通知主线程; 阻塞则指主线程在等待结果时, 不做其他事, 非阻塞指主线程在等待结果时可以做其他事; 一般采用的都是异步非阻塞才能达到不切换线程上下文, 不忙等的问题.

#### nio与bio的区别
* buffer vs stream
> stream不是缓存的, 不能移动数据, 除非进行cache, 而buffer 是一整块数据过来, 可以移动数据, 移动指针, 更加的灵活

* non blocking vs blocking
> bio 每接收到一个新的连接, 因为接收连接,再进行读写这个过程是阻塞的, 不知道什么时候数据会过来,  必须要创建一个新的线程去等待数据, 若数据没准备好, 则线程结束, 会频繁切换唤醒线程, 保存线程上下文, 造成大量的资源浪费; 而bio是主线程去接收成百上千数量的连接, 并在selector上面注册对应的事件, 当数据准备好之后, 事件驱动的方式给读写线程一个通知(操作系统提供的select/poll/epoll/kqueue 等函数功能), 让读写线程知道数据是ok的, 立马进行读写(一般是开启cpu 核心个线程数, 该过程是同步阻塞的, 占用cpu的工作, 属于memory copy, 速率在 GB/s 以上, 性能很高), nio相比于bio 适用于大量长连接的场景, 例如即时通讯软件 qq.  

#### AIO(异步非阻塞IO)

```
// 使用server上的accept方法

public abstract <A> void accept(A attachment,CompletionHandler<AsynchronousSocketChannel,? super A> handler);
CompletionHandler为回调接口，当有客户端accept之后，就做handler中的事情。

socketServer = AsynchronousServerSocketChannel.open().bind(new InetSocketAddress (PORT));
socketServer.accept(null,
                new CompletionHandler<AsynchronousSocketChannel, Object>() {
                    final ByteBuffer buffer = ByteBuffer.allocate(1024);
 
                    public void completed(AsynchronousSocketChannel result,
                            Object attachment) {
                        System.out.println(Thread.currentThread().getName());
                        Future<Integer> writeResult = null;
                        try {
                            buffer.clear();
                            result.read(buffer).get(100, TimeUnit.SECONDS);
                            buffer.flip();
                            writeResult = result.write(buffer);
                        } catch (InterruptedException | ExecutionException e) {
                            e.printStackTrace();
                        } catch (TimeoutException e) {
                            e.printStackTrace();
                        } finally {
                            try {
                                server.accept(null, this);
                                writeResult.get();
                                result.close();
                            } catch (Exception e) {
                                System.out.println(e.toString());
                            }
                        }
                    }
 
                    @Override
                    public void failed(Throwable exc, Object attachment) {
                        System.out.println("failed: " + exc);
                    }
                });
// 这里使用了Future来实现即时返回

```


### java stream

* how to gracefully close stream

1. [use "Execute Around” idiom](https://stackoverflow.com/questions/341971/what-is-the-execute-around-idiom)
2. since java7 , can use try-with
3. write a IOUtils

```
public final class IOUtil {
  private IOUtil() {}

  public static void closeQuietly(Closeable... closeables) {
    for (Closeable c : closeables) {
        if (c != null) try {
          c.close();
        } catch(Exception ex) {}
    }
  }
}
```

Then your code would be reduced to:

```
try {
  copy(in, out);
} finally {
  IOUtil.closeQuietly(in, out);
}

```
## UNIX network programming

####  6.2 IO model
![https](https://drive.google.com/uc?id=1Ie2B8Iwl61DjxYJIE9gi2ccIdFa2FjMD)

分为五种, blocking io, non blocking io, sigal driven io, io 多路复用, aio 完全异步的io.
对于一个输入操作, 主要分为两个阶段, 一是等待数据从网络到达, 二是将数据从内核态拷到用户态.

1. blocking io
两个阶段全是阻塞的

![enter image description here](https://drive.google.com/uc?id=1Ie2B8Iwl61DjxYJIE9gi2ccIdFa2FjMD)

2. non blocking io
while 循环发起recvform 调用(消耗cpu ), 询问数据是否准备好, 若没有准备好就直接返回, 数据准备好之后, 再发起系统调用去等待数据从内核态拷贝到用户态, 总之是同步非阻塞.
![enter image description here](https://drive.google.com/uc?id=1JVZTgB7uCilJdHJFnlEP4eW1B_J7cyk6)

3. io 多路复用
目前java nio 就是这个模型, select 调用阻塞, 有数据返回, 就发起系统调用去等待数据拷贝到用户态, 跟第二个模型多了一次select 系统调用, 但是不需要一直轮训获取, 减少cpu 消耗

![enter image description here](https://drive.google.com/uc?id=1fBCsvLomiJ_MP2T1ga5a9o05puj8rQCh)

4. signal driven io
注册信号发送到内核, 注册了一个回调的handler, 当有数据到达之后通知应用, 应用在发起系统调用, 同步阻塞等待数据拷贝到内核态
![enter image description here](https://drive.google.com/uc?id=1tnnPtlxvxb_vNzj-4A9avOCcxrRmtDwP)

5. AIO 
完全异步的IO, 调用aio 函数, 注册handler 到内核, 一直到内核将数据拷贝到用户态之后才通知用户态, 异步非阻塞的, 但是却不是所有的操作系统都支持

![enter image description here](https://drive.google.com/uc?id=1dfK2SGuXzaszMGewnraT9Nv9YPrq4hpE)


## netty in action 笔记

* 为什么要用netty, netty 解决了什么问题
netty 简化了网络编程, 使得网络编程变得容易, 兼容老版本, 高性能, 并支持各种协议;
要想实现高性能的网络处理, 光利用java NIO api 是容易出错, 且很工作量浩大的事, 而且很难扩展
_[The Netty project](https://netty.io/)_ is an effort to provide an asynchronous event-driven network application framework and tooling for the rapid development of maintainable high-performance · high-scalability protocol servers and clients.

Netty is an NIO client server framework that enables quick and easy development of network applications such as protocol servers and clients. It greatly simplifies and streamlines network programming such as TCP and UDP socket server development.

#### 1.3  netty core components
* channel 

an open connection to an entity such as a hardware device, a file, a
network socket, or a program component that is capable of performing
one or more distinct I/O operations, for example reading or writing.
可以把channel 理解为装载着数据的即将到来(或者离开的)车

* callback

A callback is simply a method, a reference to which has been provided to another
method. This enables the latter to call the former at an appropriate time. Callbacks
are used in a broad range of programming situations and represent one of the most
common ways to notify an interested party that an operation has completed.
可以理解为把需要执行的函数的引用传递给另一个函数, 并在适当的时机执行

Netty uses callbacks internally when handling events; when a callback is triggered
the event can be handled by an implementation of interface ChannelHandler

```
public class ConnectHandler extends ChannelInboundHandlerAdapter {
@Override
public void channelActive(ChannelHandlerContext ctx)
throws Exception {
System.out.println(
"Client " + ctx.channel().remoteAddress() + " connected");
}
}
// 这是一个channel connected的callback
```

* future

In short, the notification mechanism
provided by the ChannelFutureListener eliminates the need for manually checking
operation completion.
不同于jdk 的future, netty 提供了futureListener , 不需要轮训结果

#### echoServer  and echoClient
> All Netty servers require the following:
 At least one ChannelHandler—This component implements the server’s processing
of data received from the client—its business logic.
 Bootstrapping—This is the startup code that configures the server. At a minimum,
it binds the server to the port on which it will listen for connection requests
// 需要ChannelHandler 去处理业务逻辑, Bootstrapping 把服务器绑定到端口去监听

* echoServer
> Because your Echo server will respond to incoming messages, it will need to implement
interface ChannelInboundHandler, which defines methods for acting on inbound
events. This simple application will require only a few of these methods, so it will be
sufficient to subclass ChannelInboundHandlerAdapter, which provides a default implementation of ChannelInboundHandler // 根据到来的事件, 应该选择channelInboundHandler, 或他的子类 ChannelInboundHandlerAdapter

* echoClient
The Echo client will
1 Connect to the server
2 Send one or more messages
3 For each message, wait for and receive the same message back from the server
4 Close the connection

#### netty 新连接的建立过程
1. boss 线程检查到有新来的连接, 即accept 事件, 则将该channel 注册到worker EventLoopGroup
2. 从 worker EventLoopGroup 选择一个EventLoop 线程来关注channel 的其余事件, 例如read, write, connect, channelHandler 事件.

#### EventLoop
![enter image description here](https://drive.google.com/uc?id=1Ul-aPtMSynBht574s51DFaiOIHG4ddi2)

events 在eventloop 的逻辑执行图
![enter image description here](https://drive.google.com/uc?id=1c8Ey4mUgM0MPOU6DYRySq5eoX-y77_EV)


包括两个部分, 一是concurrent, 二是networking, 
>Netty’s EventLoop is part of a collaborative design that employs two fundamental APIs:
concurrency and networking. First, the package io.netty.util.concurrent builds
on the JDK package java.util.concurrent to provide thread executors. Second, the
classes in the package io.netty.channel extend these in order to interface with
Channel events.

* netty IO thread model

**boss EventLoopGroup** 负责accept connection, 并把channel registe to worker threads. EventLoopGroup 如果不指定构造函数, 使用的是默认的jdk ScheduledThreadPool, 线程数量是cpu 核心数*2. 
**worker EventLoopGroup** 负责执行具体的io operation, 一个channel 的所有事件, 包括connect, read, write和注册的所有 channelHandlers 都会只由一个 worker threads 执行, 避免了多线程的同步和线程切换问题, channel 和thread 的模型是: N:1, 所以一个事件的延迟会阻塞到这个channel 的其他事件和该线程其他channel 事件, 所以长时间运行的event 需要新建线程池处理, 如下图
![enter image description here](https://drive.google.com/uc?id=1WQUx6bJJBY0baCRcIRK167g80UmE4eqj)

#### Channel
![enter image description here](https://drive.google.com/uc?id=1aaIWakV2GzVZKYT0fhbKbDEJoO27GG9z)


#### ChannelFuture
在netty 中所有的IO 操作都是异步的, 

#### ChannelPipeline
* channelHandler, channelPipeline, channelHandlerContext 关系
![enter image description here](https://drive.google.com/uc?id=1siMHP8xl-eThZ8S1Z6t09eDXbdgZ1NCD)


>A ChannelPipeline provides a container for a chain of ChannelHandlers and defines
an API for propagating the flow of inbound and outbound events along the chain.
When a Channel is created, it is automatically assigned its own ChannelPipeline.
ChannelHandlers are installed in the ChannelPipeline as follows:
a. A ChannelInitializer implementation is registered with a ServerBootstrap.
b. When ChannelInitializer.initChannel() is called, the ChannelInitializer
installs a custom set of ChannelHandlers in the pipeline.
c. The ChannelInitializer removes itself from the ChannelPipeline.

#### ChannelHandler
![enter image description here](https://drive.google.com/uc?id=15_bQwwS2_OILQjbevtzfNynUsDZSXsMK)
> ChannelHandler has been designed specifically to support a broad range of uses,
and you can think of it as a generic container for any code that processes events
(including data) coming and going through the ChannelPipeline // 理解为用户业务代码和将要处理的事件和数据的容器

* sharable
A ChannelHandler intended
for this use must be annotated with @Sharable; otherwise, attempting to add it to
more than one ChannelPipeline will trigger an exception.

> WHY SHARE A CHANNELHANDLER? A common reason for installing a single
ChannelHandler in multiple ChannelPipelines is to gather statistics across
multiple Channels

* 作用
Typical uses for ChannelHandlers include:
■ Transforming data from one format to another
■ Providing notification of exceptions
■ Providing notification of a Channel becoming active or inactive
■ Providing notification when a Channel is registered with or deregistered from
an EventLoop
■ Providing notification about user-defined events

* ChannelHandlerAdapter
![enter image description here](https://drive.google.com/uc?id=1vSj4o-xRlC2oIp3rsQLQ6lV-8Rflw7ix)

#### Bootstrapping
> Netty’s bootstrap classes provide containers for the configuration of an application’s
network layer, which involves either binding a process to a given port or connecting
one process to another one running on a specified host at a specified port. // 实现连接服务器, 或者监听端口的功能

* server Bootstrapping 和client Bootstrapping的最大区别

![enter image description here](https://drive.google.com/uc?id=19D0jtFsCA3Qgo2Y2E4UKfmObzx0zyyTV)
> The second difference is perhaps more significant. Bootstrapping a client requires
only a single EventLoopGroup, but a ServerBootstrap requires two (which can be the
same instance). Why?
A server needs two distinct sets of Channels. The first set will contain a single
ServerChannel representing the server’s own listening socket, bound to a local port.
The second set will contain all of the Channels that have been created to handle incoming  client connections—one for each connection the server has accepted. Figure 3.4 illustrates this model, and shows why two distinct EventLoopGroups are required.
The EventLoopGroup associated with the ServerChannel assigns an EventLoop
that is responsible for creating Channels for incoming connection requests. Once a connection has been accepted, the second EventLoopGroup assigns an EventLoop to
its Channel. // server Bootstrapping 有两个EventLoopGroup , 一个绑定了ServerChannel  负责根据新来的connection 创建Channels , 另一个负责处理已经accepted conncetions, 

#### linux epoll
作为linux 默认的NIO API, 比java 的NIO api 更高效, 因为java 的nio 为了多平台的兼容性 做了某些妥协. 
> To substitute epoll for NIO
in that listing, replace NioEventLoopGroup with EpollEventLoopGroup and NioServer-
SocketChannel.class with EpollServerSocketChannel.class. // 使用基于epoll 的netty api

#### Dealing with a Stream-based Transport
网络传输的单位是字节, 并且接收和发送都有缓存, 有可能接收到的字节未达到一个单位消息的字节大小, 那么收到的消息就是不完全的, 所以需要额外的解码和编码. 具体见 https://netty.io/wiki/user-guide-for-4.x.html " Dealing with a Stream-based Transport", 解决方法是在CHANELPipeline 添加handler 
```
public class TimeDecoder extends ReplayingDecoder<Void> {
    @Override
    protected void decode(
            ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
        out.add(in.readBytes(4));
    }
}
```
#### netty bytebuffer
相比于jdk 原生的bytebuffer, 没有类似flip() 的方法用于读写的转变, 因为它有读写两个指针, 如图
![enter image description here](https://drive.google.com/uc?id=1kfRUR1sa27tBjPhQ_9XHffPO6JqKTi15)


* 用法
```
Charset utf8 = Charset.forName("UTF-8");
ByteBuf buf = Unpooled.copiedBuffer("Netty in Action rocks!", utf8);
System.out.println((char)buf.getByte(0));
int readerIndex = buf.readerIndex();
int writerIndex = buf.writerIndex();
buf.setByte(0, (byte)'B');
System.out.println((char)buf.getByte(0));
assert readerIndex == buf.readerIndex();
assert writerIndex == buf.writerIndex();
```
* 工厂方法
ByteBufAllocator, ByteBufHolder, Unpooled

#### TCP 拆包, 封包
* 拆包
通过看dubbo 服务端接收的源码, 大致明白了服务端本地持有一个buffer, 读剩下的数据都保存在这个buffer 里, 每次来了新的数据都先合并到这个buffer

* 如何将字节数组转化为对象呢
以dubbo decode 为例, 每次channel 读取了一部分字节, 就decode 一部分数据看看是否已经得到了完整的数据对象, 因为编码的时候会把数据长度也编码到字节流的header 中, 如果字节长度不够, 则会跳过此次轮训, 等待下次轮训的时候再次检查字节长度, 如果超过了, 则取字节长度的字节流, 剩下的保留在本地的buffer 里, 用于与下次发送的数据先进行合并.  dubbo 字节头部的编码方式如图: 
![enter image description here](https://drive.google.com/uc?id=1Af-c0cPmH_qwpaMZkEHgKVF2dvFqCWwB)
dubbo encode data header length: 16 bytes
包括: magic number, request id, 序列化的contentTypeId, 是否是异步的等. 



#### netty 性能
https://github.com/netty/netty/issues/3919

* 工具方法
ByteBufUtil, ReferenceCounted 对ByteBuffer 引用计数法回收, 
```
java -Dio.netty.leakDetectionLevel=ADVANCED 
// 可以用netty 抽样检查bytebuffer 是否有泄漏
```

### 网络协议
* ping和telnet

ping 基于ICMP 协议, 属于IP 层, telnet 基于telnet 协议, 属于TCP 层, 互相独立, 一个可通并不代表另一个可以通. 
若是需要java socket 可连, 则需要开通Aip  到Bip 的Bport, 只是单向的, 若是双向的, 则需要注明, 否则默认单向

### 疑问
* netty timeout and idle connection
* Listen socket and connected socket 
* What is file descripter
* MappedByteBuffer and directByteBuffer
* zero copy
* epoll 作为linux 默认的NIO api
* hi, [@trustin](https://github.com/trustin) i am newer to netty. I have read the above comments , but i am still not clear about current thread model. I want to make sure that all events from one channel, including channelHandlers, read and write events is always done by one eventloop from worker eventloop, and not executed by other threads, is that right ? Then, if two channels bind to the same threads, one channelHandler may block other events, is that right ? So, if there are long time events, we should use new threadpool to handle , is that right, and are there any examples to show ?  
Thank you first ! https://github.com/netty/netty/issues/1912

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTgwNTcwNzY3MywtMTEwNDQzMTQxNV19
-->