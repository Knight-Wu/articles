### 需求
领导提出打印日志性能差, 异步打印可能会丢消息等等问题, 故需要看清logback 内部实现逻辑, 采用异步或者批量的方式(自己想...)提高性能.

### 寻找思路
以下涉及的源码均是logback-1.0.13, 以下文章有利于快速了解logback
> 参考链接
1. [https://cloud.tencent.com/developer/article/1154748](https://cloud.tencent.com/developer/article/1154748)

再通过自己翻看源码, debug, 基本上掌握了logback的逻辑, 大概耗时半天, 此时也联系到公司的测试, 了解到之前他们做压测的时候, 发现**当接口逻辑简单时,tps 很高时, 打印日志将占用大量的时间, 大概有百分之七八十, 都在等待日志进入blockQueue (基于队列满, 阻塞的配置)**, 而将日志打印的一个配置: immediateFlush 改成false, 性能提升很大. 故定位到 LayoutWrappingEncoder.doEncode(E event) , 如下图, 但是此时并不知道这个flush的实现细节, 接下来参考了这篇文章, [logback.xml immediate=false 到底缓存空间是多大](http://k1280000.iteye.com/blog/2265177)
定位到是 BufferOutputStream, 此时设计方案初步明了: **基于bufferSize 和时间进行flush , 提升消费能力, 进一步提升logback的性能**但是基于如下背景: 
公司logback 版本混乱, 通过统一升级logback 版本的方式去推动, 相当困难, 目前没有建立严格的jar包审查体系, 故放弃修改源码; 采用提供独立 jar包的形式,  那么问题来了, 如何在不改动源码的情况下, 改变 BufferOutputStream 的bufferSize, 并周期性刷新? 

### 方案
经过思考和搜索, 参考这篇文章 [https://stackoverflow.com/questions/11829922/logback-file-appender-doesnt-flush-immediately](https://stackoverflow.com/questions/11829922/logback-file-appender-doesnt-flush-immediately), 提供一个新的appender, encoder, BufferOutputStream 去实现.

1. BufferOutputStream , 整合了logback的这两个类的功能, 构造函数传入参数bufferSize, 
2. BufferFileAppender, 初始化




> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTQzNTY4OTg4OV19
-->