* int(3) vs int(4)
跟storage size 并没有关系, 仍然需要4 bytes, 
但是若指定了ZERO_FILL, 则不足三位会在前面补0, 

   if the stored value has less digits than  _x_,  `ZEROFILL`  will prepend zeros.
    
    > **INT(5) ZEROFILL**  with the stored value of 32 will show  **00032**  
    > **INT(5)**  with the stored value of 32 will show  **32**  
    > **INT**  with the stored value of 32 will show  **32**
    
  if the stored value has more digits than  _x_, it will be shown as it is.
    
    > **INT(3) ZEROFILL**  with the stored value of 250000 will show  **250000**  
    > **INT(3)**  with the stored value of 250000 will show  **250000**  
    > **INT**  with the stored value of 250000 will show  **250000**

* varchar(3) vs varchar(4)
并不影响storage size, storage size 只跟存储的具体string 有关,  max storage size 仍然一样, 若是单字节编码, 大约是65535 bytes, 需要用一个字节或两个字节来存储长度, 


> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTI2OTIwNzE5MCwzNDM4NTE3NzJdfQ==
-->