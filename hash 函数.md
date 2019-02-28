
#### hashmap 8 hashcode()
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
所以问题基本明了, 因为hashmap 的数组table 的size 均为2的次方, 取模的方法是: hashcode() & (size-1), 假设size为16, size -1 低四位为1111 高位均为0 , 与hashcode() &, 则只取决于hashcode 的低位bit. 
所以特殊情况, 如何处理这种低位均是0 的hash 值: 
hashcode() 方法是无符号右移( 为了将高位移下来), 然后和原数异或(使低位和高位混淆, 保留了高位的信息,  如果是或的话, 还是低位不变), 
这个答案说的很好:
>JDK 源码中 HashMap 的 hash 方法原理是什么？ - 胖君的回答 - 知乎 https://www.zhihu.com/question/20733617/answer/111577937

#### & 替代 %
https://www.itcodemonkey.com/article/4425.html
```
  
X % 2^n = X & (2^n - 1)

2^n表示2的n次方，也就是说，一个数对2^n取模 == 一个数和(2^n - 1)做按位与运算 。

假设n为3，则2^3 = 8，表示成2进制就是1000。2^3 -1 = 7 ，即0111。

此时X & (2^3 - 1) 就相当于取X的2进制的最后三位数。

从2进制角度来看，X / 8相当于 X >> 3，即把X右移3位，此时得到了X / 8的商，而被移掉的部分(后三位)，则是X % 8，也就是余数
```

& 比 % 快很多, 前者需要5个 cpu 周期, 后者需要至少 26 个. 
https://aigo.iteye.com/blog/2292341


#### hash 算法
如何衡量一个好的hash 函数
> We have three primary requirements in implementing a good hash function for a given data type:

-   It should be  _deterministic_—equal keys must produce the same hash value.
    
-   It should be  _efficient to compute_.
    
-   It should  _uniformly distribute the keys_.

####  解决hash 冲突
https://zhuanlan.zhihu.com/p/29520044
> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTQzNzMxOTg5LDUyMDAyNTk5LDMyNTA1Mj
YwNSwxNjkyODkxODMsNTE5ODY3MDY5XX0=
-->