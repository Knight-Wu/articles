
### http 1.1 相比 http 1.0 优化的点
1. 默认 connection: keep-live, 连接被多个请求复用, 规范做法是: 客户端发送: Connection:close, 明确要求服务器关闭连接. 另外: 对于同一个域名，大多数浏览器允许同时建立6个持久连接。
2. 管道机制, 之前是 req A, resp A, req B, resp B, 后面改进之后是            reqA, req B, resp A, resp B, 但是还是先返回 A 的 resp
3. Content-Len
> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTExMTA5NTY0MDksNzk0Mjk4NTk3XX0=
-->