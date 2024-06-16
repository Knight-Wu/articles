
# zero copy 
详见: https://developer.ibm.com/articles/j-zerocopy/

如果不用 zero copy, 那么从读文件到通过网络发送需要经过以下四个步骤
1. 操作系统从磁盘读文件到内核空间的 page cache
2. 应用从内核空间的 page cache 读到用户空间的 buffer
3. 应用从用户空间的 buffer 写到内核空间的 socket buffer
4. 操作系统从内核空间的 socket buffer 读取数据写到 NIC buffer, 再网络发送

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTc0MzU4MzY5Ml19
-->
