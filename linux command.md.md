* grep and tail 
 
```
tail -n 10000 fileName| grep  "search phrase" 
只能返回

tail -nf file.name 
// 滚动获取文件的后n行
```
* 返回文件及目录中关键词的个数
```
zgrep 可以对压缩文件进行grep
grep -roh keyWord dirOrFileName | wc -w
tail -100 fileName |grep -C100 keyWord // 从fileName的后一百行找keyWord, 并显示keyWord的前后一百行
```

* 目录的大小（包括里面的所有文件）‘’
 du -sh * 

* 查找文件
    ```
    find . -mmin -10
    ```
    查找当前目录下,最近更新十分钟内的文件
    ```
    find . -name 'my*' //查找当前目录下, 以my开头的文件
    find `pwd` -name 'name' // 返回文件的绝对路径
    ```
    

    
* netstat

Displays active TCP connections, ports on which the computer is listening, Ethernet statistics, the IP routing table, IPv4 statistics (for the IP, ICMP, TCP, and UDP protocols), and IPv6 statistics (for the IPv6, ICMPv6, TCP over IPv6, and UDP over IPv6 protocols). Used without parameters, netstat displays active TCP connections.
  * Syntax

netstat [-a] [-e] [-n] [-o] [-p Protocol] [-r] [-s] [Interval]
Top of page 

  * Parameters

-a   : Displays all active TCP connections and the TCP and UDP ports on which the computer is listening.

-e   : Displays Ethernet statistics, such as the number of bytes and packets sent and received. This parameter can be combined with -s.

-n   : Displays active TCP connections, however, addresses and port numbers are expressed numerically and no attempt is made to determine names.

-o   : Displays active TCP connections and includes the process ID (PID) for each connection. You can find the application based on the PID on the Processes tab in Windows Task Manager. This parameter can be combined with -a, -n, and -p.

-p   Protocol   : Shows connections for the protocol specified by Protocol. In this case, the Protocol can be tcp, udp, tcpv6, or udpv6. If this parameter is used with -s to display statistics by protocol, Protocol can be tcp, udp, icmp, ip, tcpv6, udpv6, icmpv6, or ipv6.

-s   : Displays statistics by protocol. By default, statistics are shown for the TCP, UDP, ICMP, and IP protocols. If the IPv6 protocol for Windows XP is installed, statistics are shown for the TCP over IPv6, UDP over IPv6, ICMPv6, and IPv6 protocols. The -p parameter can be used to specify a set of protocols.

-r   : Displays the contents of the IP routing table. This is equivalent to the route print command.
Interval   : Redisplays the selected information every Interval seconds. Press CTRL+C to stop the redisplay. If this parameter is omitted, netstat prints the selected information only once.
* 常见用法

  > netstat -anp|grep :$PORT

  查看监听某端口的进程
  
  
* ps(查看进程命令)
  * ps -ef 
> To see every process on the system using standard syntax
   
  * ps -ef|grep keyword
  * ps -ef|grep tomcat
  >查看所有tomcat进程

* 查看文件大小
```
ls -lh     以mb显示文件大小
```

### 查找关键字
```
    grep keyword filepath|less 
```

> zip文件查找关键字(不需解压)
```
zipgrep keyword filename|less
```

>zip多文件查找关键字(不需解压)
```
for file in filename;do unzip -c $file|grep keyword;done|less
帮助命令: unzip -hh
```
>查找关键字出现的个数
```
grep -o keyword filename|wc -l
// wc为workcount 
// -l print the new line counts
// -o only-matching
```

* zless 不解压, 直接打开压缩文件; zgrep不用解压

* scp 在服务器间拷贝文件
> scp -r 可以递归拷贝整个目录 


* chown

     Use chown to change ownership and chmod to change rights.
    Note that both these commands just work for directories too. The -R option makes them also change the permissions for all files and directories inside of the directory.
    
    For example
    
    sudo chown -R username:group directory
    will change ownership (both user and group) of all files and directories inside of directory and directory itself.
    
    sudo chown username:group directory
    will only change the permission of the folder directory but will leave the files and folders inside the directory alone.
    
    As enzotib mentioned, you need to use sudo to change the ownership from root to yourself.
    
    Edit:
    
    Note that if you use chown <user>: <file> (Note the left-out group), it will use the default group for that user.
    
    If you want to change only the group, you can use:
    
    chown :<group> <file>
    
    
* free
> 查看机器内存使用情况
total = used+ free+ buff/cache
![1](B6A2BA2EF42C420BBB5EF95034F9F367)

* top
> 查看各进程

* sysctl 
> sysctl kernel.hostname=newHostName 修改机器的hostName, 立即生效,启动新的session后可见, 但是重启后会失效

> 还可以通过这个改: 
hostnamectl set-hostname hostname

> hostname 可以参考 http://www.cnblogs.com/kerrycode/p/3595724.html

* shutdown
> shutdown -r 重启 linux主机

* tar

> tar -cvf tarName.tar fileName 可以将目录和多个文件打包

> tar -czvf tarName.tar.gz fileName 压缩成gzip文件, 比较快, 但是压缩率不高

> tar -cjvf tarName.tar.bz2 fileName 压缩成bz2文件,比较慢, 但是压缩率
> 解压 bz2文件: bzip2 -dk filename.bz2 然后再用tar -xvf 解包

> tar -xf archive.tar -C /target/directory // 解包到任意目录

* 查看linux系统版本



>cat /etc/*-release

* yum
> yum search packageName
* yum提示another app is currently holding the yum lock;waiting for it to exit
> rm -f /var/run/yum.pid  //可以通过强制关掉yum进程


* linux命令历史记录
> history 
> ctrl+R, 然后再搜索, 直接回车


* nohup
让进程忽略hangup 信号, 而什么是hangup信号呢, 当用户注销（logout）或者网络断开时，终端会收到 HUP（hangup）信号从而关闭其所有子进程。
其他长期运行进程的命令见 [https://www.ibm.com/developerworks/cn/linux/l-cn-nohup/index.html](https://www.ibm.com/developerworks/cn/linux/l-cn-nohup/index.html)


```
nohup command & // 最简单的模式
nohup command & // 命令执行完时打出信息到shell 屏幕上
nohup command > myOutPut.out 2>&1 & // 把标准输入和输出重定向到自己的文件
```

---
#### 搭建cdh测试环境总结
1. 机器
> 五台CentOS 7.3, 内存200G左右, cpu 40多核吧, 硬盘2T, 这个机器很折腾, 因为中间需要用yum 安装软件, 但是之前用的其他操作系统, 第一次安装的型号记不得了, 第二次用的CentOS-7.5, 因为操作系统提供的package和安装软件需要的package 版本不一致, 导致中途重装了两次, 很浪费时间; 后面又发现系统盘大小很小, 数据盘没挂载, 又是一番折腾. 所以以后需要先check 机器的配置, 包括OS版本, 内存的大小, 磁盘的大小等

2. yum源
> 因为不能直连外网, 其他同事搭了一个代理服务器, 使用清华的yum源才好, 但是这个代理也很不稳定 中间经常卡住, 也不懂怎么测试, 还好后面比较熟悉yum命令, 知道可以使用yum search packageName来测试.




#### shell 单引号双引号
[http://wiki.jikexueyuan.com/project/13-questions-of-shell/double-single.html](http://wiki.jikexueyuan.com/project/13-questions-of-shell/double-single.html)

### shell 语法坑
* for循环中不能用$引用的变量
```

#!/bin/bash

upperlim=10

for i in {0..10}
do
echo $i
done

for i in {0..$upperlim} # this cant work, will print {0..10}
do
echo $i
done
#because Brace expansion occurs  **before** variables are expanded
#you can use eval and seq

for i in `eval echo {0..$d}`
do
echo $i
done

for i in $(seq $lowerlimit $upperlimit)
do
echo $i
done

```

<!--stackedit_data:
eyJoaXN0b3J5IjpbLTEzNTgxOTM5MTAsMTAxMjYxODc1MiwxMD
I4NDY2MDM5LC02MjY5NzE1OTRdfQ==
-->