
### 一. 优化磁盘mount选项
> cloudera给出的建议

 您的Datanode的磁盘mount选项有些问题， 当前bundle中不是所有datanode都收集了mount信息， 所以我只能按照有的datanode的信息来指出， 例如主机sgpd229-016， 它的磁盘mount信息如下：
 
 /dev/sdv1 /mnt/disk21 ext4 rw,relatime,stripe=64,data=ordered 0 0 /dev/sdi1 /mnt/disk8 ext4 rw,relatime,stripe=64,data=ordered 0 0 /dev/sdx1 /mnt/disk23 ext4 rw,relatime,stripe=64,data=ordered 0 0 /dev/sdw1 /mnt/disk22 ext4 rw,relatime,stripe=64,data=ordered 0 0 /dev/sdt1 /mnt/disk19 ext4 rw,relatime,stripe=64,data=ordered 0 0 /dev/sdc1 /mnt/disk2 ext4 rw,relatime,stripe=64,data=ordered 0 0 /dev/sdm1 /mnt/disk12 ext4 rw,relatime,stripe=64,data=ordered 0 0 /dev/sdk1 /mnt/disk10 ext4 rw,relatime,stripe=64,data=ordered 0 0 /dev/sdl1 /mnt/disk11 ext4 rw,relatime,stripe=64,data=ordered 0 0 /dev/sdr1 /mnt/disk17 ext4 rw,relatime,stripe=64,data=ordered 0 0 /dev/sds1 /mnt/disk18 ext4 rw,relatime,stripe=64,data=ordered 0 0 /dev/sdb1 /mnt/disk1 ext4 rw,relatime,stripe=64,data=ordered 0 0
 
 按照下面的文档 https://www.cloudera.com/documentation/enterprise/release-notes/topics/rn_consolidated_pcm.html#install_cdh_filesystem_noatime 以及下面的文档（Mount Options章节) https://www.oreilly.com/library/view/hadoop-operations/9781449327279/ch04.html
 
 
##### 总结一下: 包括以下两个内容, 一是关闭File Access Time, 二是选择异步IO
> File Access Time

Linux filesystems keep metadata that record when each file was accessed. This means that even reads result in a write to the disk. To speed up file reads, Cloudera recommends that you disable this option, called  atime, using the mount option in  /etc/fstab. When mounting data partitions, it’s best to disable both file atime and directory atime.


> async IO

However, using the  sync  option will lead to poor performance for services that write data to disks, such as HDFS, YARN, Kafka and Kudu. In CDH most writes are already replicated. Therefore, having synchronous writes to disks is unnecessary, expensive, and not worth the added safety it provides.
* 操作步骤

> 手动更改参数
```
1. cat /etc/fstab 
# 先查看当前的挂载点的选项, 默认如果是default, 等于rw,suid,dev,exec,auto,nouser,async，具体可查 man mount
# 我们目前是realtime, 解释可以参考: 
# https://www.mnstory.net/2017/08/09/file-access-time/
2. vi /etc/fstab 
# 将/dev/data/   /data/    ext3    defaults 0 0 修改为: 
# /dev/data/   /data/    ext3    noatime,nodiratime,async 0 0

3. mount -o remount /data
# 重新挂载
4. 或者可以通过命令行更改指定目录的文件选项,跟第二步是同样的效果: 
mount -o rw,noatime,nodiratime,data=ordered,async /data

```

> 使用我提供的脚本修改

```

#!/bin/sh
set -o nounset
set -o errexit 
echo 'change line $1 from $2 /etc/fstab/ '
echo 'mount disk count :$3'
# $1 和 $2 为 /etc/fstab需要修改的行数, 将 defaults 改为 defaults,noatime,nodiratime
sed '$1,$2s/defaults/defaults,noatime,nodiratime/g' /etc/fstab
sed -i '$1,$2s/defaults/defaults,noatime,nodiratime/g' /etc/fstab
for i in $(seq 1 $3)
do
   mount -o remount /app$i 
   #/app$i 为挂载路径
done

```

> 相关资料可以参考
[https://www.mnstory.net/2017/08/09/file-access-time/](https://www.mnstory.net/2017/08/09/file-access-time/)
### 二. 关闭THP
>Summary

RHEL and CentOS versions 6.2+, as well as SLES 11 SP2, include a feature called "Transparent Huge Page (THP)" compaction which interacts poorly with Hadoop workloads. This can cause a serious performance regression compared to other operating system versions on the same hardware. Workaround available.

>Symptoms

-   top  and other system monitoring tools show a large percentage of the CPU usage classified as "system CPU". If system CPU usage is 30% or more of the total CPU usage, your system may be experiencing this issue.
-   Poor performance

> Applies To

-   CDH 5
-   RHEL 6.2+
-   CentOS 6.2+
-   SLES 11 SP2

> Instructions

#### The workaround is to disable  THP  compaction:
 This can be done without rebooting by logging in as root and executing the following command:
```
$ echo never >  /sys/kernel/mm/redhat_transparent_hugepage/defrag

If you have sudo priviledges, you can also run:

$ sudo sh -c "echo 'never' > /sys/kernel/mm/redhat_transparent_hugepage/defrag"

2. To make the changes persistent, add the line below to  /etc/rc.local

echo never >  /sys/kernel/mm/redhat_transparent_hugepage/defrag

3. Confirm changes have been made:  
  
$ cat  /sys/kernel/mm/redhat_transparent_hugepage/defrag

```

### 三. 排查磁盘和网络问题
> 现象

可以在相关的Datanode中发现很多Slow的信息如下： 
```
[root@host-10-17-101-69 hdfsLog]# egrep -o "Slow.*?(took|cost)" hadoop-cmf-hdfs-DATANODE-sgpd229-021.log.out.5 |sort |uniq -c 
75 Slow BlockReceiver write data to disk cost 
1107 Slow BlockReceiver write packet to mirror took 
17 Slow flushOrSync took 
26 Slow manageWriterOsCache took 
50 Slow PacketResponder send ack to upstream took 
具体查看命令: 
egrep -o "Slow.*?(took|cost)" /path/to/current/datanode/log | sort | uniq -c
```
* 日志解释

 > Slow BlockReceiver write data to disk cost  
  
This is measured as the time taken to write to disk when a data packet comes in for a block write. Java-wise, its just the duration measurement behind an equivalent of "FileOutputStream.write(…)" call, which actually may not even target the disk in most setups, and go to the Linux buffer cache instead.

> Slow BlockReceiver write packet to mirror  
  
This measures the duration taken to write to the next DN over a regular TCP socket, and the time taken to flush the socket. We forward the same packet here (small sizes, like explained above). An increase in this typically indicates higher network latency, as Java-wise this is a pure SocketOutputStream.write() + SocketOutputStream.flush() cost.

原贴见: [https://community.cloudera.com/t5/Storage-Random-Access-HDFS/slow-blockreceiver-write-data-to-disk/td-p/29740](https://community.cloudera.com/t5/Storage-Random-Access-HDFS/slow-blockreceiver-write-data-to-disk/td-p/29740)

具体操作排查手册可参考:  [https://cloud.tencent.com/developer/article/1158307](https://cloud.tencent.com/developer/article/1158307)
 
 ### 调整配置的文件描述符的数量
<!--stackedit_data:
eyJoaXN0b3J5IjpbNDQwMjYzMjQ2XX0=
-->