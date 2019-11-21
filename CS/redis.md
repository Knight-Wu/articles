
### redis pipeline 和 transaction 的区别
pipeline 主要为了减少客户端和服务端的交互, 多个命令一起flush 到服务端, 然后执行一条返回一条命令的结果. 
而transaction 则是会缓存多条命令, 一起执行, 再一起返回结果, 中间不能插其他的命令, 

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTEwMDE5Njk3OTJdfQ==
-->