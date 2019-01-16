**用了dubbo 差不多两年, 其实很多都用过看过了, 但是就是没总结, 现在总结还不晚**

### dubbo 消费者从本地缓存获取提供者列表, 若提供者更改, 能推送更新到缓存(By programming)
> 代码调用如图: 

![](https://drive.google.com/uc?id=1KpmEBe7mhzPNlZ0zgLO-y5lMahW-Jexw)

> 我这边当初这样做的背景是

 用一个插件在每个应用打印全链路日志, 当碰到提供者特殊服务器异常, 如线程池满, 超时等异常时需要打印特殊的异常码, 以及提供者的应用名, 但是这两个异常是提供者发送信息给消费者, 然后消费者端接受到之后抛出一个RPCException, 不能通过Invoker.Url.getParameter("application")直接获取提供者应用名, 因为此时并没有真正进入提供者, 所以需要在消费者本地缓存获取. 

> 从本地缓存获取的大致逻辑

因为不能在某个提供者异常时, 消费者端大量的查询zk, 故需要先通过缓存获取, 如果提供者重启等情况导致提供者应用名改变, 缓存应该能更新. AbstractRegistry.notify 即实现了推送更新到缓存的目的.


### lsf针对dubbo 所做的改造

* 容错策略
默认的容错策略是failover, 但是经过生产测试, 服务器有问题时还是会接收到不到百分之一的请求, 并不能识别服务器真死(服务器挂掉, jvm down) 和假死((内存, cpu ,线程池等资源耗尽), 所以增加容错策略 failtrue, 
1. 准备识别服务器异常, 在一定时间内, 不重试过载的服务器.
2. 一旦有异常请求, 即开始心跳, 心跳会进入到服务端的线程池, 就像是一个正常的请求, 一段时间内心跳正常则恢复正常调用, 否则剔除异常服务器(在 AbstractClusterInvoker 的doSelect() 就直接剔除心跳失败的服务器) 


* 心跳
    因为dubbo 的默认心跳, 不进入业务线程池, 不能判断整个调用链的情况, 只能判断服务器有没有挂, 内存是否还有,  
    在服务端提供一个monitorService，返回服务器状态（cpu，内存和线程池），若服务器连续正常返回且线程池使用率低于0.95或前后两次线程池使用率下降0.1则结束心跳，心跳超时时间默认500毫秒，间隔一定周期发送一次心跳，根据服务器数量设定心跳失败次数的阈值，超过阈值则剔除对应的机器，若所有机器均异常，则一定会保留心跳失败次数最小的机器。
    * 触发条件
      * 有timeoutEx或rejectEx（新增的）
      * 根据统计信息，判断服务器某个方法的平均响应时间是之前的2.5倍
    * 问题
      * 服务端未升级高版本的jar包，导致没有monitorService，打印日志，结束心跳。
 

### dubbo的大致流程

预备知识: 
> dubbo SPI

参考了 http://cxis.me/2017/02/18/Dubbo%E4%B8%ADSPI%E6%89%A9%E5%B1%95%E6%9C%BA%E5%88%B6%E8%AF%A6%E8%A7%A3/

相比于jdk 的SPI, 不需要一次性去加载所有的实现类, 有可能有些实现类是完全不用的, 例如有些Protocol 在某些情况下肯定不用加载的, 
例如 Protocol 有多个实现类, 何时加载, 如何调用. Protocol 的实现类通过如下方式进行配置: 

![enter image description here](https://drive.google.com/uc?id=1zCSXUbTbqeVYhi135qtiaUfujTXZ5fUN)
![](https://drive.google.com/uc?id=1py584PGkFm-rjjvUfSuhB-ldvOec_YHB)

启动时将所有实现的类文件进行扫描并加载成Class 对象, 
![enter image description here](https://drive.google.com/uc?id=1s652rCYIfBrvNj211rqufmOU6VQ1LqAk)

然后手动拼装代码生成, 编译生成 AdativeClass. 
![enter image description here](https://drive.google.com/uc?id=1WFz-UZzrNiEuu32GiQeGWEfXFvp81H5k)

例如生成的Protocol$Adpative 的类文件如下, 

![enter image description here](https://drive.google.com/uc?id=1MHDklXncWwDYBzJ4rqCNw6PqxyrAysAL)

这样就可以根据url 获取相应的字段, 例如当protocol 是dubbo 时, 只实例化 DubboProtocol 这个对象, 并发起调用, 并不初始化其他Protocol .

> dubbo AOP

wrapper 类具有复制的构造函数, 例如ProtocolFilterWrapper 的构造函数是

```
ProtocolFilterWrapper(Protocol protocol){
  this.protocol = protocol;
}
```
这类具有复制构造函数的会作为AOP 去封装对应的被调用的对象(**具体如何做到 ProtocolFilterWrapper 作为被最先调用的 Protocol ?**)

> 构建调用链

初始化的时候, 在调用具体协议之前, 先调用 ProtocolFilterWrapper, ProtocolListenerWrapper 

ProtocolFilterWrapper 基于责任链模式(添加代码的时候, 不影响已有代码, 只是重新构造这个责任链 ), 构建了filter 和invoker 的调用链, 如下图所示, 从代理类的 InvocationHandler 开始, 选出此次调用的invoker(代表哪台机器), 经过多个filter 传递, 传递到dubboInvoker.doInvoke 发起远程调用.

![enter image description here](https://drive.google.com/uc?id=1q2spxJAwxa0cccftYg16qqI5_gLMBX4k)





>  服务暴露

参考了 http://jm.taobao.org/2013/11/14/3138/

直接在本机暴露服务, 先构建出url, 例如 dubbo://service-host/com.foo.FooService?version=1.0.0, 然后再把本机的服务端口打开

 向注册中心暴露服务, 构建出url : registry://registry-host/com.alibaba.dubbo.registry.RegistryService?export=URL.encode("dubbo://service-host/com.foo.FooService?version=1.0.0"), 通过url 的registry协议, 调用 RegistryProtocol 的 export()方法 将url注册到注册中心, 然后再根据dubbo协议头, 通过 DubboProtocol 的 export()将本地提供者的端口打开.

export的过程中, 首先将实现类, 例如HelloWorldImpl通过 ProxyFactory 类的 getInvoker 方法使用 ref 生成一个 AbstractProxyInvoker 实例, 从而得到Invoker, 再后, 就是将Invoker转化为 Exporter .分为两种情况
1. Dubbo的实现, 打开socket服务, 监听客户端的请求, 并返回.
2. RMI 的实现,  RMI 协议的 Invoker 转为 Exporter 发生在 RmiProtocol类的 export 方法，它通过 Spring 或 Dubbo 或 JDK 来实现 RMI 服务，通讯细节这一块由 JDK 底层来实现，这就省了不少工作量


> 服务引用

初始化: 
1. DubboNamespaceHandler 等类通过解析xml 配置文件, 去初始化ReferenceBean,  解析参数配置构造url: registry://registry-host/com.alibaba.dubbo.registry.RegistryService?refer=URL.encode("consumer://consumer-host/com.foo.FooService?version=1.0.0"),  
2. 通过RegistryProtocol 获取注册中心, 并注册消费者, 再订阅注册中心的服务(就是建立zkClient 监听服务的变化), 
3. 初始化DubboProtocol, 建立连接(默认不是lazy 的), 这样初始化invoker 就完成了.
4. 再根据invoker 创建动态代理, 根据代理去封装invoker.invoke() , 最后返回代理给spring 容器

调用: 
1. 通过Invoker 的调用链去发起调用, 所有的调用都会转发到InvokerInvocationHandler invoke 方法, 会经过cluster 和loadBalance, filter等, 最后发起请求, 等待提供者返回.


### dubbo默认的心跳
> 检测provider和consumer间的连接是否存在

* provider的心跳检测
> 默认开启, 默认60s内没有接受到任何请求, 就会开始心跳(针对所有channel 发送req), 超过三次心跳没有响应, 就断开连接, 释放资源

* consumer的心跳
> 默认60s内没有接受到任何消息就开始心跳, 超过三次没有接受到响应, 则会发起重连.

### zookeeper的作用
1. 提供者节点失效了, 与zk的会话超时, zk会知道, 主动删除节点信息
2. 提供者节点信息变更了, 由zk推送通知, 消费者能够知道, 消费者watch相关节点的变化.

### 研究dubbo 各层的作用

* dubbo-proxy
dubbo 代理层的作用一直都很模糊, 动态代理可以参考这篇文章做大致的了解, 
[https://juejin.im/post/5ad3e6b36fb9a028ba1fee6a](https://juejin.im/post/5ad3e6b36fb9a028ba1fee6a)
dubbo 代理层的作用, 见官网文档: [http://dubbo.apache.org/zh-cn/docs/dev/design.html](http://dubbo.apache.org/zh-cn/docs/dev/design.html)

> Proxy 层封装了所有接口的透明化代理，而在其它层都以 Invoker 为中心，只有到了暴露给用户使用时，才用 Proxy 将 Invoker 转成接口，或将接口实现转成 Invoker，也就是去掉 Proxy 层 RPC 是可以 Run 的，只是不那么透明，不那么看起来像调本地服务一样调远程服务

什么叫像本地服务一样调远程服务啊??? 看了代理生成的源码还是很模糊, 直到再回头看了下java 代理的作用, 看了InvocationHandler 代码的注释, dubbo 使用InvokerInvocationHandler 实现了InvocationHandler , 把对接口的调用都转化为invoker的调用, 达到的效果: spring container 服务提供者是代理对象, 对这个服务的所有方法的调用, 都会经过 InvocationHandler的invoke 方法, dubbo 加以改造, 转化为了对invoker的调用, 并把方法和参数封装到了Invocation 里面.

* java 是怎么实现把被代理类的所有方法都转移到invocationHandler 的invoke 方法的? https://docs.oracle.com/javase/8/docs/technotes/guides/reflection/proxy.html

#### 动态代理

> jdk 动态代理

1. 包：如果所代理的接口都是 public 的，那么它将被定义在顶层包（即包路径为空），如果所代理的接口中有非 public 的接口（因为接口不能被定义为 protect 或 private，所以除 public 之外就是默认的 package 访问级别），那么它将被定义在该接口所在包（假设代理了 com.ibm.developerworks 包中的某非 public 接口 A，那么新生成的代理类所在的包就是 com.ibm.developerworks），这样设计的目的是为了最大程度的保证动态代理类不会因为包管理的问题而无法被成功定义并访问；  
2. 类修饰符：该代理类具有 final 和 public 修饰符，意味着它可以被所有的类访问，但是不能被再度继承
3. 用一个weakCache 实现, key, subkey, value, 其中1,3是弱引用, 就是classloader 和代理类是作为弱引用使用, 
有两级缓存,
 一级缓存key 是classloader, 是一个 weakReference (当只被弱引用引用时, 下次gc 则被清除掉), value 为valueMap.
二级缓存, key 是classloader 和interface 接口所组成的key 对象, 由接口数量而决定, value 是代理类.


* 生成的代理类的结构
https://blog.csdn.net/mhmyqn/article/details/48474815


因为是继承了Proxy, 并implements 相应的接口, java 不能多继承, 所以只支持对接口的代理, 另外也代理了Object.java 的hashcode, toString, equal 方法, 可以对这些方法做特殊处理.
另外代理类是怎么调用委托类的方法呢, 通过反射去调用.

> cglib 代理

采用继承委托类的形式, 如果是 static方法,private方法,final方法等描述的方法是不能被代理的。默认代理Object中equals,toString,hashCode,clone等方法。比JDK代理多了clone。

> dubbo javaassist 代理

1. 生成 invoker 代理

用weakhashmap 生成, key 是 classloader的weakReference , 不会阻止classloader 被gc, 所以当classloader 被gc 之后, 这个map 里面的entry 也会被清空, 防止内存泄漏. value 是一个valueMap, key 是接口名, value 是代理类的弱引用. 
通过javaassist 生成的代理类的结构如下: 
![enter image description here](https://drive.google.com/uc?id=1Lhp2LuIa2m-fZ93pbSwfeSRXqgf6cyBr)

等于说把所有方法的调用都转化为对InvocationHandler invoke  方法的调用. 
而实际发起调用的时候, 通过Wrapper 转化为对接口提供者的调用. wrapper 在服务暴露时就初始化好
 
 wrapper 的生成如图: 
 ![enter image description here](https://drive.google.com/uc?id=1msOglqWEjCpftwgeqqslA5KyWsC1v3MZ)
 
 
#### 如何确定dubbo 线程池线程数大小




#### 序列化
*  thrift hessian 这两个rpc 框架
* Dubbo 消费者使用单一链接的方式，什么时候达到网络瓶紧
* 海量数据下的典型架构设计和性能优化之道, 精通常用架构原则

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTQzMDI2OTY0OSwxNzMzOTE1Nzc0LDEyNj
c3MzEyNjksNjQzNzcwNjIsLTEwODAzOTk5NTAsLTE1OTQ1MzA0
ODIsLTE3OTgxNDA4NjEsLTIwMTY2MjkyNzQsLTEwOTQwOTA2Mz
IsNzY1MTQwOTQsMTY1NTMwODE1OSwtMTc1NTIwMTAwMCwxMjM5
ODAxMzg1LDE4MDQ0ODkzMzAsLTE5NTA0MjQwNDQsLTEwMTc4Nz
E5OTgsLTE4NzE0MDE4ODksLTE4ODY1OTM3NDMsLTIxMzE5MDEx
MDAsODAzMjMxNDZdfQ==
-->