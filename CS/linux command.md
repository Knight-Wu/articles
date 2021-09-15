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
* 目录大小排序
du -h --max-depth=1 | sort -h

* 查找文件
    ```
    find . -mmin -10
    ```
    查找当前目录下,最近更新十分钟内的文件
    ```
    find . -name 'my*' //查找当前目录及子目录下, 以my开头的文件
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
* 根据 port 查看 pid

  > netstat -ltnp|grep :$PORT

-   `l`  – tells netstat to only show listening sockets.
-   `t`  – tells it to display tcp connections.
-   `n`  – instructs it show numerical addresses.
-   `p`  – enables showing of the process ID and the process name
  
 * 根据 pid 查看占用的 port
```
lsof -Pan -p PID -i

```
  
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

   * grep keyword filepath|less 
   * grep -rni  'keyword' 'path'  查找当前目录及子目录, 包含 keyword 的文件. 
   `-r`  or  `-R`  is recursive,
   `-n`  is line number,
-i case-unsensitive



> zip文件查找关键字(不需解压)
```
zipgrep keyword filename|less
```

>zip多文件查找关键字(不需解压)
```
for file in filename;do unzip -c $file|grep keyword;done|less
帮助命令: unzip -hh
```
* 解压jar 文件
```
jar xvf test.jar

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
 查看各进程资源
top -Hbp processId // 查看这个进程的所有线程, 输入的pId 实际是线程id,
https://blog.csdn.net/flysqrlboy/article/details/79314521

* sysctl 
> sysctl kernel.hostname=newHostName 修改机器的hostName, 立即生效,启动新的session后可见, 但是重启后会失效

> 还可以通过这个改: 
hostnamectl set-hostname hostname

> hostname 可以参考 http://www.cnblogs.com/kerrycode/p/3595724.html

* shutdown
> shutdown -r 重启 linux主机

* lrzsz
rz // 上传文件, win 直接弹出对话框, win 需要支持Zmodem
sz // 下载文件, 
* tar

> tar -cvf tarName.tar fileName 可以将目录和多个文件打包

> tar -czvf tarName.tar.gz fileName 压缩成gzip文件, 比较快, 但是压缩率不高

> tar -cjvf tarName.tar.bz2 fileName 压缩成bz2文件,比较慢, 但是压缩率
> 解压 bz2文件: bzip2 -dk filename.bz2 然后再用tar -xvf 解包

> tar -xf archive.tar -C /target/directory // 解包到任意目录

> tar zxvf FileName.tar.gz  // 解压 tar.gz 文件
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
程序执行完成之后, nohup 启动的进程会退出, 不需要显示的在程序末尾 : exit 0


```
nohup command & // 最简单的模式
nohup command & // 命令执行完时打出信息到shell 屏幕上
nohup command > myOutPut.out 2>&1 & // 把stdout 和stderr 重定向到自己的文件

nohup command > /dev/null 2>&1 & // 不输出到 nohup.out
```

但是ssh 登录之后, 当 ssh session 被中断, 有可能之前启动的 background process 会被退出. 
This problem can also be overcome by redirecting all three I/O streams:
```
nohup myprogram > stdout 2>&1 < /dev/null &
```
详细解释: 
[http://www.snailbook.com/faq/background-jobs.auto.html](http://www.snailbook.com/faq/background-jobs.auto.html)
如果链接失效可以看 当前目录: SSH FAQ
[https://stackoverflow.com/questions/29142/getting-ssh-to-execute-a-command-in-the-background-on-target-machine](https://stackoverflow.com/questions/29142/getting-ssh-to-execute-a-command-in-the-background-on-target-machine)

* set
```
set +x; command; set -x // 可以将每一行执行的详细命令都解析到屏幕上输出
```

* 执行脚本
可参考 https://www.cnblogs.com/pcat/p/5467188.html
```
1. 在当前目录下执行
./shellscript  


2. Execute using sh/ bash interpreter

$ sh scriptfile

$ bash scriptfile


3. executes the commands specified in the scriptfile in the current shell, In other words, this  prepares the environment for you.
$ . ./scriptfile

 “dot space dot slash” Usage Example:

Typically we use this method, anytime we change something in the .bashrc or .bash_profile. i.e After changing the .bashrc or .bash_profile we can either logout and login for the changes to take place (or) use “dot space dot slash” to execute .bashrc or .bash_profile for the changes to take effect without logout and login.

$ cd ~

$ . ./.bashrc

$ . ./.bash_profile

4. Execute Shell Script Using Source Command, 和3 是一样的效果

The builtin source command is synonym for the . (dot) explained above. If you are not comfortable with the “dot space dot slash” method, then you can use source command as shown below, as both are same.

$ source ~/.bashrc

```

* system service operation

systemctl usage

The complication ends here. In fact, the stopping|starting|restarting of services on Linux is now quite simple. Let's say we're on CentOS and we want to stop the Apache server. To do this we'd open up a terminal window and issue the command:
```
sudo systemctl stop httpd

The Apache server would stop and you'd be returned to the bash prompt. To start the same service, we'd issue the command:

sudo systemctl start httpd

The service would start and you'd be returned to your bash prompt.

To restart the same service, we'd issue the command:

sudo systemctl restart httpd

The service would restart and you'd be returned to the bash prompt.

The above commands can be run on CentOS, Ubuntu, Redhat, Fedora, Debian, and many more
```


* 给某个用户, 赋予操作某个文件的权限
You can use chown and chgrp commands to change the owner or the group of a particular file or directory.

```
# ls -lart tmpfile
-rw-r--r-- 1 himanshu family 0 2012-05-22 20:03 tmpfile

# chown root tmpfile

# ls -l tmpfile
-rw-r--r-- 1 root family 0 ``2012-05-22 20:03 tmpfile
```

* 查询系统默认config 值
You can find out a system's default page size by querying its configuration via the  `getconf`command:
```
$ getconf PAGE_SIZE
4096
```

### ssh
* 多台机器免密登录
1. 在用户目录下, 生成公钥和私钥. 
 ssh-keygen -t rsa
 2. 将用户公钥 .ssh/id_rsa.pub 粘贴到另一台机器的 .ssh/authorized_keys, 例如 A 要 ssh 登录 B, 则将 A 的公钥粘贴到 B 的authorized_keys, 也可以用 ssh-copy-id 命令, 过程就是: 所谓"公钥登录"，原理很简单，就是用户将自己的公钥储存在远程主机上。登录的时候，远程主机会向用户发送一段随机字符串，用户用自己的私钥加密后，再发回来。远程主机用事先储存的公钥进行解密，如果成功，就证明用户是可信的，直接允许登录shell，不再要求密码。

 
 * ssh_all_host.sh 操作多台机器
```
#! /bin/bash

command=$1

filename=cluster_host

while IFS= read -r line; do

echo $line

ssh -n $line $command 

done < "$filename"
```
* ssh -n 
Redirects stdin from /dev/null (actually, prevents reading from stdin), 意思就是 send an eof to any read call from that process, For example, when starting a background process remotely over ssh, you [must redirect stdin](https://serverfault.com/a/36436) to prevent the ssh waiting for remote process input.

here is an example proves ssh eat stdin. 
```

#!/bin/bash

echo -e "host1\nhost2\nhost3" |
while read host; do
  echo "ssh to $host"
  ssh $host "cat"
done

We’ll get the output like

ssh to host1
host2
host3
```

"host2\nhost3" was never sent to the while loop but eaten by ssh



* iptables
当 telnet 某个端口不通的时候, 检查一下目标机器的 iptables
sudo iptables  -A INPUT -s 10.129.0.0/16 -p tcp -m tcp --dport 9300 -j ACCEPT;

* ssh

> ssh -NfL localport:targetAddr:targetPort remoteIp

只要 remoteIp 通过 telnet targetAddr:targetPort , 并不需要只在targetAddr

* output hex code from keyboard input
> xxd -psg

再按键, 回车结尾, 末尾的 : 0a 是ascii newline (LF)

---
#### 搭建cdh测试环境总结
1. 机器
> 五台CentOS 7.3, 内存200G左右, cpu 40多核吧, 硬盘2T, 这个机器很折腾, 因为中间需要用yum 安装软件, 但是之前用的其他操作系统, 第一次安装的型号记不得了, 第二次用的CentOS-7.5, 因为操作系统提供的package和安装软件需要的package 版本不一致, 导致中途重装了两次, 很浪费时间; 后面又发现系统盘大小很小, 数据盘没挂载, 又是一番折腾. 所以以后需要先check 机器的配置, 包括OS版本, 内存的大小, 磁盘的大小等

2. yum源
> 因为不能直连外网, 其他同事搭了一个代理服务器, 使用清华的yum源才好, 但是这个代理也很不稳定 中间经常卡住, 也不懂怎么测试, 还好后面比较熟悉yum命令, 知道可以使用yum search packageName来测试.



#### shell google 编程指南
https://fangpeishi.com/google_shell_style_guide.html
[https://google.github.io/styleguide/shell.xml](https://google.github.io/styleguide/shell.xml)

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

### linux hard limit, such as file descriptor limit
>$ **ulimit -aH**
core file size (blocks)       unlimited
data seg size (kbytes)        unlimited
file size (blocks)            unlimited
max locked memory (kbytes)    unlimited
max memory size (kbytes)      unlimited
open files                    1024
pipe size (512 bytes)         8
stack size (kbytes)           unlimited
cpu time (seconds)            unlimited
max user processes            4094
virtual memory (kbytes)       unlimited

### List File Opened By a PID
``` 
lsof -p pid  
lsof -a -p pid
 ```
OR  
```
cd /proc/pid/fd  
ls -l | less 
You can count open file, enter:   ls -l | wc -l
```

### Count All Open File Handles
> lsof | wc -l

###  Empty File Content by Redirecting to Null
```  
echo > fileName
```

### monitor disk usage

```
dstat -cd --disk-util --disk-tps
```

### check ssd or hdd
```
cat /sys/block/sda/queue/rotational
```
You should get  `1`  for hard disks and  `0` for a SSD.

```
lsblk -d -o name,rota
```
where  `ROTA`  means  `rotational device`(hdd)  (`1`  if true,  `0`  if false) 

### check raid conf
<!--stackedit_data:
eyJoaXN0b3J5IjpbNTYyNzI1MzQ2LC03ODQ5ODAwMjEsLTM0NT
YzOTM3MSwxNDMwNDA3MjMzLDE2OTk2NTE3ODAsLTE3NDMzNDE4
OTIsLTk4MzU5MzQ4NiwxMTMyNTc0MTk1LC03OTYyNjIzNSwtMT
Y5NTkwOTczOSwzMzEzOTUyNzMsNjgxMDU1MzQwLDk4NzU5NzE2
OCwyMTIxNDg4NjYwLDgwODYyNDExMCwtMTQ2MzQxMTIwNSwxOT
M5MjE5MzI3LC00MDk4ODg4MTUsLTE4OTU2NDY5ODcsNTIxNTM5
NzUyXX0=
-->