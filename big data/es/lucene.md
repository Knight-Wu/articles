# lucene 用法图
![image](https://user-images.githubusercontent.com/20329409/220824791-229315c5-c028-448e-be9e-f66bd43e875e.png)
```
　　结合代码说明一下四个步骤：

IndexWriter iw=new IndexWriter();//创建IndexWriter
Document doc=new Document( new StringField("name", "Donald Trump", Field.Store.YES)); //构建索引文档
iw.addDocument(doc);            //做索引库
IndexReader reader = DirectoryReader.open(FSDirectory.open(new File(index)));
IndexSearcher searcher = new IndexSearcher(reader); //打开索引
Query query = parser.parse("name:trump");//解析查询

TopDocs results =searcher.search(query, 100);//检索并取回前100个文档号
for(ScoreDoc hit:results.hits)
{
    Document doc=searcher .doc(hit.doc)//真正取文档
}
```
# FST 在索引上与其他数据结构的比较
参考：
1. https://www.cnblogs.com/sessionbest/articles/8689030.html
2. https://zhuanlan.zhihu.com/p/35814539

![image](https://user-images.githubusercontent.com/20329409/220824893-1f407eca-206d-4da3-b4f1-a119ec882c96.png)

其中词典结构尤为重要，有很多种词典结构，各有各的优缺点，最简单如排序数组，通过二分查找来检索数据，更快的有哈希表，磁盘查找有B树、B+树，但一个能支持TB级数据的倒排索引结构需要在时间和空间上有个平衡，下图列了一些常见词典的优缺点：
![image](https://user-images.githubusercontent.com/20329409/220825197-3a15e2fe-24f0-452f-90df-08731c9b099e.png)
</br>
## mysql innodb B+ tree
理论基础：平衡多路查找树
    优点：外存索引、可更新
    缺点：空间大、速度不够快
![image](https://user-images.githubusercontent.com/20329409/220825216-3103017c-ad4d-48a4-856c-a411a267f09b.png)
</br>
## 跳跃表
  优点：结构简单、跳跃间隔、级数可控，Lucene3.0之前使用的也是跳跃表结构，后换成了FST，但跳跃表在Lucene其他地方还有应用如倒排表合并和文档号索引。
    缺点：模糊查询支持不好
![image](https://user-images.githubusercontent.com/20329409/220825287-f0e4c00a-7927-4351-b3d9-294e54a69b0a.png)

</br>

## FST
  
  ![image](https://user-images.githubusercontent.com/20329409/220825385-e3f46ec2-6c56-4386-ad5e-8d4ec2a172d0.png)
理论基础:   《Direct construction of minimal acyclic subsequential transducers》，通过输入有序字符串构建最小有向无环图。
优点：内存占用率低，压缩率一般在3倍~20倍之间、模糊查询支持好、查询快
缺点：结构复杂、输入要求有序、更新不易
Lucene里有个FST的实现，从对外接口上看，它跟Map结构很相似，有查找，有迭代：
```
String inputs={"abc","abd","acf","acg"}; //keys
long outputs={1,3,5,7};                  //values
FST<Long> fst=new FST<>();
for(int i=0;i<inputs.length;i++)
{
    fst.add(inputs[i],outputs[i])
}
//get 
Long value=fst.get("abd");               //得到3
//迭代
BytesRefFSTEnum<Long> iterator=new BytesRefFSTEnum<>(fst);
while(iterator.next!=null){...}
```
</br>
数据结构	HashMap	TreeMap	FST
构建时间(ms)	185	500	1512
查询所有key(ms)	106	218	890

　　可以看出，FST性能基本跟HaspMap差距不大，但FST有个不可比拟的优势就是占用内存小，只有HashMap10分之一左右，这对大数据规模检索是至关重要的，毕竟速度再快放不进内存也是没用的。
　　因此一个合格的词典结构要求有：
　　1. 查询速度。
　　2. 内存占用。
　　3. 内存+磁盘结合。
　　后面我们将解析Lucene索引结构，重点从Lucene的FST实现特点来阐述这三点。
# 如何解释fst 呢

目的是根据词去寻找docId，然后对docId 求并集交集等集合运算。
常见的方法就是用hashmap 去存，key 是词，value 是对应的docId 列表，但是数据量很大的情况下就很消耗空间，
</br>
lucene 的做法是用三个数据结构去表示，分别是term index， term dictionary，id（posting） skip list。
</br>
term index 理解为字典树，但是会做前缀和后缀的压缩，然后查询某个词的时间复杂度从hashMap 的O（1）变为这个词的长度O（n），因为要遍历这个词，遍历之后得到term 的id list 地址。

## id list 中如何快速查找这个id 呢
常见的是二分查找，lucene 是采用skip list，就是把数据进行分层，每层有几个索引，类似多叉树的查找方式，降低了树的深度，
### SkipList有以下几个特征
1. 元素排序的，对应到我们的倒排链，lucene是按照docid进行排序，从小到大。
2. 跳跃有一个固定的间隔，这个是需要建立SkipList的时候指定好，例如下图以间隔是3
3. SkipList的层次，这个是指整个SkipList有几层

![image](https://user-images.githubusercontent.com/20329409/220820989-e70f9ae4-b868-4734-a56b-e5d3984e2182.png)


有了这个SkipList以后比如我们要查找docid=12，原来可能需要一个个扫原始链表，1，2，3，5，7，8，10，12。有了SkipList以后先访问第一层看到是然后大于12，进入第0层走到3，8，发现15大于12，然后进入原链表的8继续向下经过10和12。
</br>
### fst 分为前缀和不用前缀两种方式
前缀: 
例如 32个词使用一个前缀, term index 树形结构最后得到的是前缀, val 是词典块的地址, 然后找到词典块, 再二分查找得到具体词，然后就可以找到id list。

不用前缀: term index 树形结构最后得到的是整个完整的词, val 是词的id list 对应的文件偏移, 就没有词典里面二分查找的过程了. 但是term index 大小会比用前缀大很多. 

term index 在内存中可以理解为 term -> dictionary address 的sortedmap, 但是由于是树形结构 key 的前缀和后缀是共用的, 所以内存要少得多. 但是查找的时候效率就不是 O(1) 了, 是 O(term 的长度). 
term dictionary 在内存中理解为 term -> chunkid, 在磁盘中term 会前缀压缩. 

下图中就是前缀的表示：
![image](https://user-images.githubusercontent.com/20329409/220820606-709b3d3b-b9e6-4af3-b381-e1787482d2f1.png)

* 还有如何进行id 列表的合并呢


# lucene 文件类型


## term dictionary
.tim 文件, 可通过 term 找到 docId 以及 term 的元数据(例如 term 在 doc 中的词频; 
dictionary 以 block 的形式组织

## term index
.tip 文件, 实际上就是 index to the term dictionary, 具体结构见: https://blog.mikemccandless.com/2010/12/using-finite-state-transducers-in.html,
每一个 field 都会产生一个独立 FST , 
FST 相当于一个 SortedMap<ByteSequence,SomeOutput>, key 是 term 的前缀, val 是block 在 disk 的位置, block 即 term dictionary 提到的.  

## term frequencies
.doc 文件, 保存着 term 对应的 doc 列表, 以及 term 出现的频率. 
