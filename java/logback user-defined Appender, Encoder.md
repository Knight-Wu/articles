### 需求
领导提出打印日志性能差, 异步打印可能会丢消息等等问题, 故需要看清logback 内部实现逻辑, 采用异步或者批量的方式(自己想...)提高性能.

### 寻找思路
以下涉及的源码均是logback-1.0.13, 以下文章有利于快速了解logback
> 参考链接
1. [https://cloud.tencent.com/developer/article/1154748](https://cloud.tencent.com/developer/article/1154748)

再通过自己翻看源码, debug, 基本上掌握了logback的逻辑, 大概耗时半天, 此时也联系到公司的测试, 了解到之前他们做压测的时候, 发现**当接口逻辑简单时,tps 很高时, 打印日志将占用大量的时间, 大概有百分之七八十, 都在等待日志进入blockQueue (基于队列满, 阻塞的配置)**, 而将日志打印的一个配置: immediateFlush 改成false, 性能提升很大. 故定位到 LayoutWrappingEncoder.doEncode(E event) ,
![enter image description here](https://drive.google.com/uc?id=1YK4-VblwCicba7XCplX2OmBzti4-1XW8)
但是此时并不知道这个flush的实现细节, 接下来参考了这篇文章, [logback.xml immediate=false 到底缓存空间是多大](http://k1280000.iteye.com/blog/2265177)
定位到是 BufferOutputStream, 此时设计方案初步明了: **基于bufferSize 和时间进行flush , 提升消费能力, 进一步提升logback的性能**但是基于如下背景: 
公司logback 版本混乱, 通过统一升级logback 版本的方式去推动, 相当困难, 目前没有建立严格的jar包审查体系, 故放弃修改源码; 采用提供独立 jar包的形式,  那么问题来了, 如何在不改动源码的情况下, 改变 BufferOutputStream 的bufferSize, 并周期性刷新? 

### 方案
经过思考和搜索, 参考这篇文章 [https://stackoverflow.com/questions/11829922/logback-file-appender-doesnt-flush-immediately](https://stackoverflow.com/questions/11829922/logback-file-appender-doesnt-flush-immediately), 提供一个新的appender, encoder, BufferOutputStream 去实现.

1. 自定义 outputStream 继承java.io.OutputStream , 整合了logback的这两个类的功能, 构造函数传入参数 bufferSize, 

2. 自定义 appender 继承RollingFileAppender , 初始化BufferOutputStream 
![enter image description here](https://drive.google.com/uc?id=1yA923Us6R5DW4VkF4PKIJQHTFUlj9T2v)
3. 自定义 encoder, 继承自EncoderBase, 整合了PatternLayoutEncoderBase和LayoutWrappingEncoder的功能, 

![](https://drive.google.com/uc?id=1-B3bpZIFiTPgS-m9tImlxRZYgm_kMsoP)
4. 最后配置文件如下, 
![enter image description here](https://drive.google.com/uc?id=1ZbecJjVla4PSqrvfZ1msbh9lL_qiGfmc)
### 碰到的问题
大概前后花了一天半的时间完成整个任务, 包括测试, 还是网上资料给力, 提供了很好的思路, 剩下的就是编码细节, 搞清encoder 的初始化逻辑等, 自定义的类需要整合哪些类的功能等问题了

* 一开始碰到一个设置immediateFlush 不生效的问题, 如下图,
![enter image description here](https://drive.google.com/uc?id=1oZxx0e7zRq_VP2NIkDZRzl0mB7w8KGom)
 在appender 和patternLayout里面设置immediateFlush 不生效, 发现设置immediateFlush=false的 encoder和接下来执行write的encoder不是一个对象, 跟着源码进行debug, 发现这里新建了一个encoder, 
![enter image description here](https://drive.google.com/uc?id=1SJ73FAADDJ4KbOd7NdjyboXd85UDnLIe)
logback也是推荐encoder 而不是layout, 所以配置里面不应该配置PatternLayout, 应该直接配置encoder, 具体原因没研究
* I/O concept flush vs sync 
 其中还发现了这个帖子, [I/O concept flush vs sync](https://stackoverflow.com/questions/4072878/i-o-concept-flush-vs-sync), 可以记录一下, 个人的理解是flush 只是基于file 这个类将buffer 刷新到文件系统缓存, 但是文件系统的缓存persist into disk 需要调用sync 
 
 * 2018-12-28 update
logback-1.0.13 没有实现shutdownhook, 自己加了一个, 跟1.3.0的hook 是一个逻辑, 只要能获取到 ContextAwareBase 的context 实例, 调用stop 方法就可以了.

### 疑问
* 如果后续压测还是性能提升较小的话, 如何提升
* logback 性能测试 https://github.com/ceki/logback-perf
* logback v_1.3.0-alpha4 版本 AsyncAppenderBase 的worker thread 为什么只用一个thread, 用多个会不会有提升?
* logback-v_1.3.0 OutputStreamAppender line 217 在加锁前面会不会有问题
* logback-v_1.3.0 OutputStreamAppender  line 138 写footer 不强制flush 会不会导致丢失?

### 总结

* 之前感觉不改源码, 直接改BufferOutputStream的初始化几乎不可能, 又不是spring有容器进行管理beans, 后续的解决思路等于是加上了一层逻辑层去重写底层的逻辑, 想到以前听到过一个计算机大神说的话, 大意是: 在计算机世界里, 没有什么加上一层逻辑层解决不了的, 想到之前写dubbo 的filter, SPI真是很方便的加入用户自己的插件.
* 测试机器，日志盘不要和数据盘在一起，这样性能也会下降很厉害
* 更加深刻理解作用域的限制, 源码里面非public的成员, 有时候影响很大.


> by the way, 开源真爽, 可以学习别人的思路, 还可以加入自己的理解.




> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTg0ODM5Mjg4Nyw2NjE3Mjc5MDldfQ==
-->