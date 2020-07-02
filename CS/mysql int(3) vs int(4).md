* int(3) vs int(4)
跟storage size 并没有关系, 仍然需要4 bytes, 
但是若指定了ZERO_FILL, 则不足三位会在前面补0, 
例如 100, 0100

* varchar(3) vs varchar(4)
并不影响storage size, max storage size 仍然一样, 若是单字节编码, 大约是65535 bytes, 


> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMzQzODUxNzcyXX0=
-->