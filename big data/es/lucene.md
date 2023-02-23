
# 如何解释fst 呢

目的是根据词去寻找docId，然后对docId 求并集交集等集合运算。
常见的方法就是用hashmap 去存，key 是词，value 是对应的docId 列表，但是数据量很大的情况下就很消耗空间，
lucene 的做法是用三个数据结构去表示，分别是term index， term dictionary，id（posting） skip list。
term index 理解为字典树，但是会做前缀和后缀的压缩，然后查询某个词的时间复杂度从hashMap 的O（1）变为这个词的长度O（n），因为要遍历这个词，遍历之后得到term 的id list 地址。

* id list 中如何快速查找这个id 呢
常见的是二分查找，lucene 是采用skip list，就是把数据进行分层，每层有几个索引，类似多叉树的查找方式，降低了树的深度，
SkipList有以下几个特征：
1. 元素排序的，对应到我们的倒排链，lucene是按照docid进行排序，从小到大。
2. 跳跃有一个固定的间隔，这个是需要建立SkipList的时候指定好，例如下图以间隔是3
3. SkipList的层次，这个是指整个SkipList有几层

![image](https://user-images.githubusercontent.com/20329409/220820989-e70f9ae4-b868-4734-a56b-e5d3984e2182.png)


有了这个SkipList以后比如我们要查找docid=12，原来可能需要一个个扫原始链表，1，2，3，5，7，8，10，12。有了SkipList以后先访问第一层看到是然后大于12，进入第0层走到3，8，发现15大于12，然后进入原链表的8继续向下经过10和12。

fst 分为前缀和不用前缀两种方式: 
前缀: 
例如 32个词使用一个前缀, term index 树形结构最后得到的是前缀, val 是词典块的地址, 然后找到词典块, 再二分查找得到具体词，然后就可以找到id list。

不用前缀: term index 树形结构最后得到的是整个完整的词, val 是词的id list 对应的文件偏移, 就没有词典里面二分查找的过程了. 但是term index 大小会比用前缀大很多. 

term index 在内存中可以理解为 term -> dictionary address 的sortedmap, 但是由于是树形结构 key 的前缀和后缀是共用的, 所以内存要少得多. 但是查找的时候效率就不是 O(1) 了, 是 O(term 的长度). 
term dictionary 在内存中理解为 term -> chunkid, 在磁盘中term 会前缀压缩. 

下图中就是前缀的表示：
![image](https://user-images.githubusercontent.com/20329409/220820606-709b3d3b-b9e6-4af3-b381-e1787482d2f1.png)

* 还有如何进行id 列表的合并呢


## 文件类型


### term dictionary
.tim 文件, 可通过 term 找到 docId 以及 term 的元数据(例如 term 在 doc 中的词频; 
dictionary 以 block 的形式组织

### term index
.tip 文件, 实际上就是 index to the term dictionary, 具体结构见: https://blog.mikemccandless.com/2010/12/using-finite-state-transducers-in.html,
每一个 field 都会产生一个独立 FST , 
FST 相当于一个 SortedMap<ByteSequence,SomeOutput>, key 是 term 的前缀, val 是block 在 disk 的位置, block 即 term dictionary 提到的.  

### term frequencies
.doc 文件, 保存着 term 对应的 doc 列表, 以及 term 出现的频率. 
