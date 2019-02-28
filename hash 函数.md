
一开始在这个链接: https://algs4.cs.princeton.edu/34hash/
看到
> _Floating-point numbers._ If the keys are real numbers between 0 and 1, we might just multiply by M and round off to the nearest integer to get an index between 0 and M-1. Although it is intuitive, this approach is defective because it gives more weight to the most significant bits of the keys; the least significant bits play no role. One way to address this situation is to use modular hashing on the binary representation of the key (this is what Java does).

对这句话不太理解:  
>because it gives more weight to the most significant bits of the keys; the least significant bits play no role.

想到之前看hashmap, hashcode() 函数注释的时候也有: 
> 
> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTIwOTMyMzk1MzddfQ==
-->