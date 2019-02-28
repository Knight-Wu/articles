
一开始在这个链接: https://algs4.cs.princeton.edu/34hash/
看到
> _Floating-point numbers._ If the keys are real numbers between 0 and 1, we might just multiply by M and round off to the nearest integer to get an index between 0 and M-1. Although it is intuitive, this approach is defective because it gives more weight to the most significant bits of the keys; the least significant bits play no role. One way to address this situation is to use modular hashing on the binary representation of the key (this is what Java does).

对这句话不太理解:  
>because it gives more weight to the most significant bits of the keys; the least significant bits play no role.

想到之前看hashmap, hashcode() 函数注释的时候也看不懂: 
> Because the table uses power-of-two masking, sets of  
 hashes that vary only in bits above the current mask will  
 always collide. (Among known examples are sets of Float keys  
 holding consecutive whole numbers in small tables.)
 
 找到这个帖子发现了问题: https://stackoverflow.com/questions/36848151/hash-codes-for-floats-in-java, 上面hashcode 的注释说的一句: 众所周知的一个例子是float 作为key 时, 连续的float 出现在小表中, 末尾会出现大量的零, 
 例如
 ```

System.out.format("%x\n", Float.floatToIntBits(1));
System.out.format("%x\n", Float.floatToIntBits(-1));
System.out.format("%x\n", Float.floatToIntBits(3));
System.out.format("%x\n", Float.floatToIntBits(-3));


```
3f800000
bf800000
40400000
c0400000
```
 ```
所以问题基本明了, 如何处理这种低位均是0 的hash 值, hashcode() 方法是无符号右移( 为了将高位移下来), 然后和原数异或(使低位和高位混淆, 如果是或的话, 还是低位不变), 

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTEzODIzMTE4MjJdfQ==
-->