# 参考资料
1. https://www.cnblogs.com/sessionbest/articles/8689030.html
2. https://zhuanlan.zhihu.com/p/35814539
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
# FST 倒排索引
## 简单概括
整体图示:
![image](https://github.com/Knight-Wu/articles/assets/20329409/1f4275ca-32bb-463c-ad26-f316267d6d7f)

 倒排索引意思是根据词找这个词所在的 docId, 叫做 fst, 简要说来就是一个内存和磁盘结合的 hashmap, key 和 val 都做了压缩, 适用与索引大小大于内存的场景, 例如几十个 GB, 那么使用 fst 就合适,
 </br>
FST，它的特点就是：
　　1、词查找复杂度为O(len(str))
　　2、共享前缀、节省空间
　　3、内存存放前缀索引、磁盘存放后缀词块
　　这跟我们前面说到的词典结构三要素是一致的：1. 查询速度。2. 内存占用。3. 内存+磁盘结合。我们往索引库里插入四个单词abd、abe、acf、acg,看看它的索引文件内容。

![image](https://github.com/Knight-Wu/articles/assets/20329409/31799322-67b4-44e7-ad90-11bf35a689aa)

</br>
　　tip部分，就是 term index, 词典的索引, 每列(对应到 es 就是每个需要 indexed 的 field )一个FST索引，所以会有多个FST，每个FST存放前缀和后缀块指针，这里前缀就为a、ab、ac。
 </br> tim里面存放后缀块和词的其他信息如倒排表(docId list, es 叫做 posting list, 在内存中的实现为跳表, 方便查找和合并)指针、TFDF等.
 </br> doc文件里就为每个单词的倒排表。
 </br>
　　所以它的检索过程分为三个步骤：
　　1. 内存加载tip文件就是term index，通过FST在内存中匹配前缀找到后缀词块位置。
  
　　2. 根据词块位置，读取磁盘中term dictionary 文件中后缀块并找到后缀和相应的posting list位置信息。
  
　　3. 根据posting list 位置去doc文件中加载具体的数据. 
  </br>
　　这里就会有两个问题，第一就是前缀如何计算，第二就是后缀如何写磁盘并通过FST定位，下面将描述下Lucene构建FST过程:
  </br>
　　已知FST要求输入有序，所以Lucene会将解析出来的文档单词预先排序，然后构建FST，我们假设输入为abd,abd,acf,acg，那么整个构建过程如下：
![image](https://github.com/Knight-Wu/articles/assets/20329409/765dd943-358d-42bd-8949-49ef44ed2923)


1. 插入abd时，没有输出。
   
2. 插入abe时，计算出前缀ab，但此时不知道后续还不会有其他以ab为前缀的词，所以此时无输出。

3. 插入acf时，因为是有序的，知道不会再有ab前缀的词了，这时就可以写tip和tim了，tim中写入后缀词块d、e和它们的倒排表位置ip_d,ip_e，tip中写入a，b和以ab为前缀的后缀词块位置(真实情况下会写入更多信息如词频等)。

4. 插入acg时，计算出和acf共享前缀ac，这时输入已经结束，所有数据写入磁盘。tim中写入后缀词块f、g和相对应的倒排表位置，tip中写入c和以ac为前缀的后缀词块位置。

</br>
以上是一个简化过程，Lucene的FST实现的主要优化策略有：

1. 最小后缀数。Lucene对写入tip的前缀有个最小后缀数要求，默认25，这时为了进一步减少内存使用。如果按照25的后缀数，那么就不存在ab、ac前缀，将只有一个跟节点，abd、abe、acf、acg将都作为后缀存在tim文件中。我们的10g的一个索引库，索引内存消耗只占20M左右。

2. 前缀计算基于byte，而不是char，这样可以减少后缀数，防止后缀数太多，影响性能。如对宇(e9 b8 a2)、守(e9 b8 a3)、安(e9 b8 a4)这三个汉字，FST构建出来，不是只有根节点，三个汉字为后缀，而是从unicode码出发，以e9、b8为前缀，a2、a3、a4为后缀，如下图：

![image](https://github.com/Knight-Wu/articles/assets/20329409/b7dce0e6-1002-4228-b4a8-2867e5f4cb3d)

## 数据结构
以下内容基本来自: https://lucene.apache.org/core/9_9_1/core/org/apache/lucene/codecs/lucene90/blocktree/Lucene90BlockTreeTermsWriter.html

### term index

![image](https://github.com/Knight-Wu/articles/assets/20329409/ca832d5f-66c6-4bfc-ab8e-abc679df48d1)

term index is the .tip file, term index 相当于 term dictionary 的 index, 会加载进内存中, 用于决定这个 term 存不存在, 不存在就不需要做磁盘查找了. 
每一个 field 都会产生一个独立 FST , FST 相当于一个 Map, key 是 term 的前缀, val 是term dictionary 在 disk 的位置.

### term dictionary

![image](https://github.com/Knight-Wu/articles/assets/20329409/40d79a87-4864-40e8-b6d2-ccbc86829f61)

前面说到 term index 的 val 就是这个 term dictionary 的文件位置, term dictionary 里面分 block 存储, 每个 block 词的数量, 就是 entry 的数量(by default 25-48), 设置为 1 就是不共享前缀, 如果超过设置的词的数量, 就会有指针指向下一个 block 的地址. 找到对应的词之后就可以找到 posting list 的文件位置, 再去磁盘查找 posting list 文件, 应该是对的, 但是这个过程和 Term Metadata 的结构感觉对不上.
![image](https://github.com/Knight-Wu/articles/assets/20329409/40ae7917-949b-4990-b724-320d8092563f)


### Term Metadata
![image](https://github.com/Knight-Wu/articles/assets/20329409/b5ffca4c-f7fe-43c0-99ff-a2049777be48)

The .tmd file contains the list of term metadata (such as FST index metadata) and field level statistics (such as sum of total term freq).

## posting list 中如何快速查找这个id 呢
常见的是二分查找，lucene 是采用skip list，就是把数据进行分层，每层有几个索引，类似多叉树的查找方式，降低了树的深度，
### posting list 在磁盘中如何压缩
1. 数据压缩，可以看下图怎么将6个数字从原先的24bytes压缩到7bytes。
![image](https://github.com/Knight-Wu/articles/assets/20329409/f7093d21-7054-4362-a199-bdb6b6feeacb)

### posting list 如何合并
跳跃表加速合并，因为布尔查询时，and 和or 操作都需要合并倒排表，这时就需要快速定位相同文档号，所以利用跳跃表来进行相同文档号查找。
这部分可参考ElasticSearch的一篇博客，里面有一些性能测试：https://www.elastic.co/blog/frame-of-reference-and-roaring-bitmaps
简单合并的例子:

假如我们的查询条件是name = “Alice”，那么按照之前的介绍，首先在term字典中定位是否存在这个term，如果存在的话进入这个term的倒排链，并根据参数设定返回分页返回结果即可。这类查询，在数据库中使用二级索引也是可以满足，那lucene的优势在哪呢。假如我们有多个条件，例如我们需要按名字或者年龄单独查询，也需要进行组合 name = "Alice" and age = "18"的查询，那么使用传统二级索引方案，你可能需要建立两张索引表，然后分别查询结果后进行合并，这样如果age = 18的结果过多的话，查询合并会很耗时。那么在lucene这两个倒排链是怎么合并呢。
假如我们有下面三个倒排链需要进行合并。

![image](https://github.com/Knight-Wu/articles/assets/20329409/c3d9f751-db5a-4ec4-9e3c-921c0be06e87)

在lucene中会采用下列顺序进行合并：
```
1. 在termA开始遍历，得到第一个元素docId=1, currentDocId = 1
2. Set currentDocId
3. 在termB中 search(currentDocId) = 1 (返回大于等于currentDocId的一个doc),

因为currentDocId == 1，继续
如果currentDocId 和返回的不相等，设置currentDocId 为最短postling list 的下一个 docId, 执行2，然后继续
如果到termC后依然符合，返回结果
并且设置 : currentDocId = termC的nextItem
然后继续步骤3 依次循环。直到某个倒排链到末尾。
```
整个合并步骤我可以发现，如果某个链很短，会大幅减少比对次数，并且由于SkipList结构的存在，在某个倒排中定位某个docid的速度会比较快不需要一个个遍历。可以很快的返回最终的结果。从倒排的定位，查询，合并整个流程组成了lucene的查询过程，和传统数据库的索引相比，lucene合并过程中的优化减少了读取数据的IO，倒排合并的灵活性也解决了传统索引较难支持多条件查询的问题。

### SkipList有以下几个特征
1. 元素排序的，对应到我们的倒排链，lucene是按照docid进行排序，从小到大。
2. 跳跃有一个固定的间隔，这个是需要建立SkipList的时候指定好，例如下图以间隔是3
3. SkipList的层次，这个是指整个SkipList有几层

![image](https://user-images.githubusercontent.com/20329409/220820989-e70f9ae4-b868-4734-a56b-e5d3984e2182.png)


有了这个SkipList以后比如我们要查找docid=12，原来可能需要一个个扫原始链表，1，2，3，5，7，8，10，12。有了SkipList以后先访问第一层看到是然后大于12，进入第0层走到3，8，发现15大于12，然后进入原链表的8继续向下经过10和12。
</br>
## FST 在索引上与其他数据结构的比较


![image](https://user-images.githubusercontent.com/20329409/220824893-1f407eca-206d-4da3-b4f1-a119ec882c96.png)

其中词典结构尤为重要，有很多种词典结构，各有各的优缺点，最简单如排序数组，通过二分查找来检索数据，更快的有哈希表，磁盘查找有B树、B+树，但一个能支持TB级数据的倒排索引结构需要在时间和空间上有个平衡，下图列了一些常见词典的优缺点：
![image](https://user-images.githubusercontent.com/20329409/220825197-3a15e2fe-24f0-452f-90df-08731c9b099e.png)
</br>
### mysql innodb B+ tree
理论基础：平衡多路查找树
    优点：外存索引、可更新,
    缺点：空间大、更新速度不够快
![image](https://user-images.githubusercontent.com/20329409/220825216-3103017c-ad4d-48a4-856c-a411a267f09b.png)
</br>
### 跳跃表
优点：结构简单、跳跃间隔、级数可控，Lucene3.0之前使用的也是跳跃表结构，后换成了FST，但跳跃表在Lucene其他地方还有应用如倒排表合并和文档号索引。
</br>
缺点：模糊查询支持不好
![image](https://user-images.githubusercontent.com/20329409/220825287-f0e4c00a-7927-4351-b3d9-294e54a69b0a.png)

</br>

### FST
  
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
```
数据结构	       HashMap	   TreeMap	  FST
构建时间(ms)	   185	       500	      1512
查询所有key(ms)	106	       218	      890
```
　　可以看出，FST性能基本跟HaspMap差距不大，但FST有个不可比拟的优势就是占用内存小，只有HashMap10分之一左右，这对大数据规模检索是至关重要的，毕竟速度再快放不进内存也是没用的。</br>
　　因此一个合格的词典结构要求有：
　　1. 查询速度。
　　2. 内存占用。
　　3. 内存+磁盘结合。
　　后面我们将解析Lucene索引结构，重点从Lucene的FST实现特点来阐述这三点。
