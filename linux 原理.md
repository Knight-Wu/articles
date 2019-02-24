
### linux 内存

* swap 
先简单根据博客资料: https://cloud.tencent.com/developer/article/1200032, 是一块为了满足虚拟内存的需求, 开辟的磁盘区域, 用来保存内存中不经常使用的数据, 根据LRU 算法.

* 什么时候开始使用swap


* buffer and cache
buffers是用来缓冲块设备做的，它只记录文件系统的元数据（metadata）以及 tracking in-flight pages，而cached是用来给文件做缓冲。更通俗一点说：buffers主要用来存放目录里面有什么内容，文件的属性以及权限等等。而cached直接用来记忆我们打开过的文件和程序。

为了验证我们的结论是否正确，可以通过vi打开一个非常大的文件，看看cached的变化，然后再次vi这个文件，感觉一下两次打开的速度有何异同，是不是第二次打开的速度明显快于第一次呢？ 接着执行下面的命令：

find /* -name  *.conf

看看buffers的值是否变化，然后重复执行find命令，看看两次显示速度有何不同。

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTI4ODk2NDc2NywxNDc1NDM5OTE3LDE5Mj
A3NDAxNTBdfQ==
-->