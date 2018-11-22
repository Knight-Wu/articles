**用了dubbo 差不多两年, 其实很多都用过看过了, 但是就是没总结, 现在总结还不晚**

### dubbo 消费者从本地缓存获取提供者列表, 若提供者更改, 能推送更新到缓存(By programming)
> 代码调用如图: 

![](https://drive.google.com/uc?id=1KpmEBe7mhzPNlZ0zgLO-y5lMahW-Jexw)

> 我这边当初这样做的背景是

 用一个插件在每个应用打印全链路日志, 当碰到提供者特殊服务器异常, 如线程池满, 超时等异常时需要打印特殊的异常码, 以及提供者的应用名, 但是这两个异常是提供者发送信息给消费者, 然后消费者端接受到之后抛出一个RPCException, 不能通过Invoker.Url.getParameter("application")直接获取提供者应用名, 因为此时并没有真正进入提供者, 所以需要在消费者本地缓存获取. 

> 从本地缓存获取的大致逻辑

因为不能在某个提供者异常时, 消费者端大量的查询zk, 故需要先通过缓存获取, 如果提供者重启等情况导致提供者应用名改变, 缓存应该能更新. AbstractRegistry.notify 即实现了推送更新到缓存的目的.




> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE0NjQxMTUzMywtMzYxMTQxNzA5LC0xMT
k0Njk3MzJdfQ==
-->