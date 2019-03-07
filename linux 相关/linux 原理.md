
### linux 内存## linux kernel development 3rd notes
### 3. process management
所有进程均是init 进程(pid=1)的子进程
* 线程
Linux has a unique implementation of threads.To the Linux kernel, there is no concept of a thread. Linux implements all threads as standard processes.The Linux kernel does not provide any special scheduling semantics or data structures to represent threads. Instead, a thread is merely a process that shares certain resources with other processes.(线程只是抽象的概念, 在内核的角度来看只是与其他进程共享资源的一种特殊的进程. )
Each thread has a unique task_struct and appears to the kernel as a normal process.
threads just happen to share resources, such as an address space, with other processes.
#### process creation
分为两个步骤, fork() 和exec()
* fork()
 creates a child process that is a copy of the current task. It differs from the parent only in its PID (which is unique), its PPID (parent’s PID, which is set to the original process),
 linux 的fork 不同于传统的fork, 是基于copy on write, 将copy 延迟到真正写入开始的时候, 如果fork() 之后马上exec(), 就压根不需要copy. 
*  fork的开销
is the duplication of the parent’s page tables and the creation of a unique process descriptor for the child
* exec()
 loads a newexecutable into the address space and begins executing it.
### 12. memory management
#### pages
Most 32-bit architectures have 4KB pages, whereas most 64-bit architectures have 8KB pages.This implies that on a machine with 4KB pages and 1GB of memory, physical memory is divided into 262,144 distinct pages.
The kernel represents every physical page on the system with a  struct page structure.
The important point to understand is that the page structure is associated with physical pages, not virtual page.

#### zones
由于硬件的限制, 所有的page 不能被一视同仁. 需要分为几个zones . 实际的分区跟计算机的体系结构相关, 在此不需要深入, 可以当做几乎所有内存都是可以被使用的.

#### slab layer
可以简要理解: free list 作为一个缓存经常使用的数据结构的一个空闲列表, 而slab layer 是内核用来维护这个free list, slab 把不同的数据结构分为不同的组, 例如一组用来存放进程描述符.

### 15. process address space
#### process address space
* memory descriptor
The kernel represents a process’s address space with a data structure called the memory descriptor.This structure contains all the information related to the process address space.
* page table
将虚拟地址转换为物理的设备, 页表分为多级, 每个进程都有一个页表.


####  swap

先简单根据博客资料: https://cloud.tencent.com/developer/article/1200032, 是一块为了满足虚拟内存的需求, 开辟的磁盘区域, 用来保存内存中不经常使用的数据, 根据LRU 算法; 也为了能让内存容下超出物理内存大小的数据. 

* 什么时候开始使用swap

```
cat /proc/sys/vm/swappiness

60
```
上面这个60代表物理内存在使用60%的时候才会使用swap

**swappiness=0的时候表示最大限度使用物理内存，然后才是 swap空间，**

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
* 如何释放内存和swap 
https://cloud.tencent.com/developer/article/1200032

* swap 由process 的使用情况
> To find out the amount of swap space used by every process, run `top` (not `htop`), press 'f' to select columns (f for fields) to display, press 'p' to add swap to display, press 'o' to sort the table (o for order by) and press 'p' again to order by swap usage.

* buffer and cache
buffer (缓冲区)起到流量整形的作用, 将多次的小io 累积成少次的大io, 减少响应的开销, 并且让io 的速率稳定. 
cache (缓存)是为了处理高速和低速设备之间的速度的不匹配(例如cpu 和memory), 通过让存储系统分级来减小这种差异带来的影响.  缓存的速度比主存快很多, 数据先从缓存取, 实际情况中cpu 都能从缓存中找到大部分数据.
### 疑问
* swap 为什么会增大

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTUzMzQwMzM4NywtMjA5NDA4MzU0OSwxNT
E2ODE3MDk3LC05OTkyMzEyMDAsMTMyODY4MjU1OSwtODQ2NTI3
MzYxLDE0MzU2MTI3OTQsMTYzMTk4NDQ2NCwtMTM0NzM0OTM0Mi
wxOTIwMTYyNDYsLTc5OTk5NDA5MywtMTUzMTQyMDUyMiwxNjk1
MjY5MDldfQ==
-->