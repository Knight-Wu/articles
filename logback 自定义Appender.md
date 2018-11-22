### 需求
领导提出打印日志性能差, 异步打印可能会丢消息等等问题, 故需要看清logback 内部实现逻辑, 采用异步或者批量的方式(自己想...)提高性能.

### 初探logback,  寻找思路
以下涉及的源码均是logback-1.0.13, 以下文章有利于快速了解logback
> 参考链接
1. [https://cloud.tencent.com/developer/article/1154748](https://cloud.tencent.com/developer/article/1154748)

再通过自己翻看源码, debug, 基本上掌握了logback的逻辑, 大概耗时半天, 此时也联系到公司的测试, 了解到之前他们做压测的时候, 将日志打印的一个配置: immediateFlush 改成false, 性能提升很大. 故定位到 LayoutWrappingEncoder.doEncode(E event) , 如下图, 但是此时并不知道这个flush的实现细节, 接下来参考了这篇文章, [logback.xml immediate=false 到底缓存空间是多大](http://k1280000.iteye.com/blog/2265177)
定位到是 BufferOutputStream, 此时设计方案初步明了. 基于如下背景: 




> 背景
公司logback 版本混乱, 通过统一升级logback 版本的方式去推动, 相当困难, 目前没有建立严格的jar包审查体系, 故放弃修改源码



> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTI1MzMwODYwNF19
-->