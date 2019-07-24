
### 问题
1. methods 和function 的区别
2. 短赋值的场景
3. 为什么要有指针
* Choosing a value or pointer receiver

There are two reasons to use a pointer receiver.

The first is so that the method can modify the value that its receiver points to.

The second is to avoid copying the value on each method call. This can be more efficient if the receiver is a large struct, for example.
4. 为什么要用make
5. interface 在 go里面怎么理解
6.   evaluation of `f`, `x`, `y`, and `z` happens in the current goroutine and the execution of `f` happens in the new goroutine ？ two gorountine ？
7. 如何理解channel
8. sync.Mutex 是控制程序执行的顺序还是内存可见性
9. go 编译和运行？ 如何引用其他文件的代码
10. go get 如何在 ide get 的时候自动指定版本 
11. go build 为什么需要 sudo , 然后再服务器执行就 exec format error 了, go install 是可以的
12. ssh 端口转发
> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTIwMzg4MTcwMDksNjcyNTE5MzAxLDEwMj
U1OTAzNTIsLTExODA4MDQ5MTQsMTk5MTc4MDMzMiwzNjAzNzI5
MDksLTE4NjAxOTUxNzIsMjYwMDgxMDA4LC0xNzU3Mzk4MjAsLT
MxMzQyNTIxNl19
-->