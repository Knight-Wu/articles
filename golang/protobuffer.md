### 好处
1. 用Base 128 Varints 编码数字

例如300 , int, 本来需要四个字节, 用这个编码之后只需要两个字节. 
让比较小的int 用更少的字节, 但是较大的数就需要更多的字节, 例如


> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTIwMjU0ODQwMTldfQ==
-->