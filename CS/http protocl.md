
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
resp 需要按次序返回, 容易造成dui'tou'yong'du
> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTMwNzEwNzcwOCw1MDcwMzkxOTYsLTExMT
QwNzkyOTUsNzk0Mjk4NTk3XX0=
-->