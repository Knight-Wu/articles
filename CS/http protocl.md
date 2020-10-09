
### http 1.1 相比 http 1.0 优化的点
1. 默认 connection: keep-live
 连接被多个请求复用, 规范做法是: 客户端发送: Connection:close, 明确要求服务器关闭连接. 另外: 对于同一个域名，大多数浏览器允许同时建立6个持久连接。
2. 管道机制
之前是 req A, resp A, req B, resp B, 后面改进之后是            reqA, req B, resp A, resp B, 但是还是先返回 A 的 resp
3. Content-Length 
 一个tcp 连接可以发送多个 resp, 就需要区分数据包是属于哪个 resp 
4. 分块传输编码
使用 content-length 的前提是, 必须知道返回的数据长度, 意味着服务器要等所有操作完成才能发送数据, 太慢了, 1.1 规定可以不使用 content-length, 而使用分块传输编码, 只要请求或响应, 头信息有 Transfer-Encoding, 则表明resp 将由数量未定的数据包组成. 

* 缺点
resp 需要按次序返回, 容易造成队头拥堵

### http 2 相比 http 1.1 的优点
1. 二进制协议
http1.1 头部必须是 ascii, 数据体可以是二进制; http 2 头部和数据体都可以是二进制, 好处在哪? 

2. 多工(Multiplexing)
请求和响应不需要按照顺序一一对应, 如果 server 发送 respA 的过程非常耗时, 可以先发送 respA 已经好的部分, 然后发送 respB, 再发送 resp A
3. 数据流
http2 将某个请求或响应的所有数据包称作一个数据流(stream), 由某个 ID 指定, 客户端发送的 ID 均是奇数, 服务器的则是偶数; 数据流发送到一半的时候, 双方都可以发送信号 (RST_STREAM frame) 来取消这个数据流, http 2 之前要取消某个数据流只能断开连接, 在可以不断开了

4.头信息压缩
可以压缩后在发送头信息, 并且可以同时维护一张头信息表, 并生成一个索引 id, 后面发送的信息只需要发送索引 id 即可
5. 服务器推送
服务器可以预测客户端接下来的行为来事先发送接下来要请求的消息
 

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTIzODIyMjU5LDUwNzAzOTE5NiwtMTExND
A3OTI5NSw3OTQyOTg1OTddfQ==
-->