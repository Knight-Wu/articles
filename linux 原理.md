
### linux 内存

* swap 
先简单根据博客资料: https://cloud.tencent.com/developer/article/1200032, 是一块为了满足虚拟内存的需求, 开辟的磁盘区域, 用来保存内存中不经常使用的数据, 根据LRU 算法.

* 什么时候开始使用swap

```
cat /proc/sys/vm/swappiness

60
```
上面这个60代表物理内存在使用60%的时候才会使用swap

swappiness=0的时候表示最大限度使用物理内存，然后才是 swap空间，

swappiness＝100的时候表示积极的使用swap分区，并且把内存上的数据及时的搬运到swap空间里面。

通常情况下：

swap分区设置建议是内存的两倍 （内存小于等于4G时），如果内存大于4G，swap只要比内存大就行。另外尽量的将swappiness调低，这样系统的性能会更好。

* 修改swappiness参数

临时性修改：
```
 sysctl vm.swappiness=10

vm.swappiness = 10

 cat /proc/sys/vm/swappiness

10

永久性修改：

vim /etc/sysctl.conf

加入参数：

vm.swappiness = 35

然后在直接：

 sysctl -p

查看是否生效：

cat /proc/sys/vm/swappiness

35
```

* buffer and cache
buffers是用来缓冲块设备做的，它只记录文件系统的元数据（metadata）以及 tracking in-flight pages，而cached是用来给文件做缓冲。更通俗一点说：buffers主要用来存放目录里面有什么内容，文件的属性以及权限等等。而cached直接用来记忆我们打开过的文件和程序。

为了验证我们的结论是否正确，可以通过vi打开一个非常大的文件，看看cached的变化，然后再次vi这个文件，感觉一下两次打开的速度有何异同，是不是第二次打开的速度明显快于第一次呢？ 接着执行下面的命令：

find /* -name  *.conf

看看buffers的值是否变化，然后重复执行find命令，看看两次显示速度有何不同。

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTIwMTIzNzE3ODEsMTQ3NTQzOTkxNywxOT
IwNzQwMTUwXX0=
-->