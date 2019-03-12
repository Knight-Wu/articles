
## linux kernel development 3rd notes
### 3. process management
所有进程均是init 进程(pid=1)的子进程
* 线程
Linux has a unique implementation of threads.To the Linux kernel, there is no concept of a thread. Linux implements all threads as standard processes.The Linux kernel does not provide any special scheduling semantics or data structures to represent threads. Instead, a thread is merely a process that shares certain resources with other processes.(线程只是抽象的概念, 在内核的角度来看只是与其他进程共享资源的一种特殊的进程. )
Each thread has a unique task_struct and appears to the kernel as a normal process.
threads just happen to share resources, such as an address space, with other processes.
#### user and   kernel space
Because a process in the UNIX system can execute in two modes, kernel or user, it uses a separate stack for each mode. When a system call is made, a _trap_ instruction is executed which causes an _interrupt_ which makes the hardware switch to kernel mode. The kernel stack of a process is null when the process executes in user mode(硬件在用户态和内核态的切换)
#### process creation
分为两个步骤, fork() 和exec()
* fork()
 creates a child process that is a copy of the current process. It differs from the parent only in its PID (which is unique), its PPID (parent’s PID, which is set to the original process),
 linux 的fork 不同于传统的fork, 是基于copy on write, 将copy 延迟到真正写入开始的时候, 如果fork() 之后马上exec(), 就压根不需要copy. 
*  fork的开销
is the duplication of the parent’s page tables and the creation of a unique process descriptor for the child
* exec()
The kernel loads an executable file into memory during an _exec_ system call, and the loaded process consists of at least three parts, called _regions_: text, data, and stack
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
* 进程的虚拟内存空间
分为数据区,代码区, 栈区, 堆区和未使用区.
代码区中存放应用程序的机 器代码，运行过程中代码不能被修改，具有只读和固定大小的特点。数据区中存放了应用程序中的全局数据，静态数据和一些常量字符串等，其大小也是固定的。堆 是运行时程序动态申请的空间，属于程序运行时直接申请、释放的内存资源。栈区用来存放函数的传入参数、临时变量，以及返回地址等数据。未使用区是分配新内 存空间的预备区域



####  swap

先简单根据博客资料: https://cloud.tencent.com/developer/article/1200032, 是一块为了满足虚拟内存的需求, 为了能让内存容下超出物理内存大小的数据,  当物理内存不够用的情况下, 根据LRU算法把最近最少使用的内存swap out.

* 什么时候开始使用swap

```
cat /proc/sys/vm/swappiness

60
```
上面这个60代表物理内存在使用超过40%的时候, 根据LRU 算法, 把不经常使用的内存swap out.

**swappiness=0的时候表示最大限度使用物理内存，然后才是 swap空间，**

swappiness＝100的时候表示积极的使用swap分区，并且把内存上的数据及时的搬运到swap空间里面。
如下图
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

* buffer
处于文件系统和磁盘间的缓冲, 写的时候变成批量写, 减少写的的频率, 数据结构可以理解为一个双端列表, 当需要一个free bufer 的时候, 会从头取; 当kernel return buffer, usually attach buffer to the tail, 那么越靠近head 的buffer 就是越不经常使用的. 

#### write dirty page
当内存中的page cache 被修改之后, 这个page 就是dirty page. 
dirty page 可以让多个dirty page 可以被一起写入同一个磁盘扇区, 提供了延迟写, 因为写操作的挂起通常不会引起应用阻塞, 但是读操作会, 这样就能提供读多写少的服务.  但是dirty page 可能会引起数据的丢失. 

* 什么时候被写到磁盘
一定时间后, 或脏页太多, page cache 太大,或调用sync(),fsync(),fdatasync( ) 强制写回.

### the block I/O layer
#### block device
The smallest addressable unit on a block device is a sector. Sectors come in various powers of two, but 512 bytes is the most common size.the device cannot address or operate on a unit smaller than the sector.

* memory page, fs block , disk sector
Sectors, the smallest addressable unit to the device, are sometimes called “hard sectors” or “device blocks.” Meanwhile, blocks, the smallest addressable unit to the filesystem, are sometimes referred to as “filesystem blocks” or “I/O blocks.” block size 必须不能大于page size, 并且要是sector 的2的次方倍


## The Design of the UNIX Operating System notes
### Introduction to the Kernel

#### Architecture of the UNIX Operating System
####  An Overview of the File Subsystem
The kernel contains two other data structures, the file tableand the user file descriptor table. The file table is a **global kernel structure, but the user file descriptor table is allocated per process**.When a process _opens or creates a file, the kernel allocates an entry from each table, corresponding to the file's inode. Entries in the three structures  user file descriptor table, file table, and inode table (maintain the state of the file and the user's access to it.) The file table keeps track of the byte offset in the file where the user's next read or write will start, and the access rights allowed to the opening process. The user file descriptor table identifier all open files for a process.

### 疑问
* swap 为什么会增大

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTg1NDExMTc1OSwtODA1MDg5ODQxLC0xNT
Y2Mzg4MDkwLDE2NzA0OTEyMTUsLTkzNDM1MDI0LDE1OTEyNTg1
NDksLTEzMzY3NTM0NDQsLTExNzc1OTE0MjksLTM4MDQ5MTM2MS
wxNjM1NTY5MTMyLC0xMjI3NTk1NDU3LC0yMjE3MTU5OSw4NjQ2
NDM0MzYsMTUzMzQwMzM4NywtMjA5NDA4MzU0OSwxNTE2ODE3MD
k3LC05OTkyMzEyMDAsMTMyODY4MjU1OSwtODQ2NTI3MzYxLDE0
MzU2MTI3OTRdfQ==
-->