## 文件类型


### term dictionary
.tim 文件, 可通过 term 找到 docId 以及 term 的元数据(例如 term 在 doc 中的词频; 
dictionary 以 block 的形式组织

### term index
.tip 文件, 实际上就是 index to the term dictionary, 具体结构见: https://blog.mikemccandless.com/2010/12/using-finite-state-transducers-in.html,
每一个 field 都会产生一个独立 FST , 
FST 相当于一个 SortedMap<ByteSequence,SomeOutput>, key 是 term 的前缀, val 是block 在 disk 的位置, block 即 term dictionary 提到的.  

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE2ODM4MDcyNzQsMjE0NzM3MzgyMSwtMT
c2MTExNjg4MV19
-->