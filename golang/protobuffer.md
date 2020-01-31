### 好处
1. 用Base 128 Varints 编码数字

例如300 , int, 本来需要四个字节, 用这个编码之后只需要两个字节. 
让比较小的int 用更少的字节, 但是较大的int 就需要五个的字节, 从统计学意义上是能节省空间的, 

具体原理: 
用每一个字节的第一位(是否是1 ) 来表示是否还有下一个字节, 所以每个字节的有效位数只有7 位. 

如果用不到 1 个字节，那么最高有效位设为 0 ，如下面这个例子，1 用一个字节就可以表示，0000 0001. 

	
> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTMwNTMxNTg2LDEyOTc0MDk2OTBdfQ==
-->