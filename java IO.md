
---

#### java IO的历史
从一开始的Socket BIO, 同步阻塞, 等待连接和读写的时候是阻塞的, 只能忙等,并且一个连接必须开多一个线程去接受否则会因为一个客户端慢导致其他客户端受影响; 若开多个线程会导致线程切换开销大, 无法应对海量连接; 到1.4的 NIO, 同步非阻塞, 连接和读写请求非阻塞, 单线程采用selector去接受请求, 再根据事件的不同用不同的线程去处理, 但是读写,创建连接仍然是同步的, 需要等待读写,连接的完成才能进行业务处理; 再到 1.7的NIO2(也可以叫做 AIO ,异步非阻塞IO), 则是读写以及连接完了通知下一步处理, 采用回调函数进行业务处理, 完全异步的形式, 适用于读写过程长的任务, 不用等待读写完成.




#### 服务器单线程处理
> 可能一个客户端慢导致其他客户端受影响
    
#### 服务器多线程处理
*问题* 
 1.1 可以开启cpu核心数个线程进行并发处理, 但是如何处理后续海量连接等问题...

#### nio处理, 主线程根据selector获取到不同的事件(包括连接, 写和读)
最简单的Reactor模式：注册所有感兴趣的事件处理器，单线程轮询选择就绪事件，执行事件处理器。
1. 事件分发器，利用selector 单线程选择就绪的事件。下面例子中的主线程
2. I/O处理器，包括connect、read、write等，这种纯CPU操作，一般开启CPU核心个线程就可以。
3. 后续处理的业务线程，在处理完I/O后，业务一般还会有自己的业务逻辑，有的还会有其他的阻塞I/O，如DB操作，RPC等。只要有阻塞，就需要单独的线程,否则会拖慢第二步的io处理.
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
缓冲区, byteBuffer看看源码很简单, 有position, mark, limit, capacity来控制字节数组读写的位置, 读之前需要flip() 回到数组头部开始读.

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
![enter image description here](![https](https://drive.google.com/uc?id=1Ie2B8Iwl61DjxYJIE9gi2ccIdFa2FjMD)://drive.google.com/uc?id=1PPqk_KLN34g5aPNV6nCTHHU-SuaHCI8H)

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

4. 
> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTExMTkwMjU0ODcsMTM0NTAzODcwLC0xOT
kwODE2ODMwLC0xMTE1ODE1NjQ5LDg4MDgzMzk0MSwxOTkxNTcy
Nzg3LC0xNjM5NDAzOTE1XX0=
-->