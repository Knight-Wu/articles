# 何为关系型

数据被组织成关系，每个关系是数据中行的无序集合，
在数据库和在编程内存中或者磁盘上只是数据模型的不同，orm就是object relation mapping 用作数据模型的映射

下图很好的表达了关系的意思, 通过id 去关联
![image](https://user-images.githubusercontent.com/20329409/212015949-352ac261-8f80-4091-805e-0cac41991d40.png)

* NOSQL 

指的是not only sql
# Tips
* 存储城市等字符串信息时，为什么要加个id呢，因为像编码一样更容易修改，id像一个引用，最终文本只存在一个地方，只修改一个地方; 防止重复; 方便编码, 用id 即可存储在一个国家的所有城市, 表示这层信息

* 多对多关系 

增加一张中间表. 例如学生和课程

* 兼容

向后兼容(back compatibility): 指的是新的代码能兼容旧的数据. 
向前兼容(forward compatibility): 旧代码能兼容新数据, 主语都是代码或程序

* 数据库的查询优化器类似于得出一个遍历数据的路径


在关系数据库中，查询优化器自动决定查询的哪些部分以哪个顺序执行，以及使用哪些索
引。这些选择实际上是“访问路径”，但最大的区别在于它们是由查询优化器自动生成的，
如下图:
![image](https://user-images.githubusercontent.com/20329409/212016529-7bae9bde-84ae-4cd5-9fa8-3ac5aaa3201e.png)

* 文档数据库和关系型数据库的区别

但是，在表示多对一和多对多的关系时，关系数据库和文档数据库并没有根本的不同：在这
两种情况下，相关项目都被一个唯一的标识符引用，这个标识符在关系模型中被称为外键，
在文档模型中称为文档引用; 但是文档数据库不利于多对多的表达, 会有很多冗余, 关系型数据库用id 来表示减少了很多存储. 

* 文档数据库和关系型数据库的融合


如果一个数据库能够处理类似文档的数据，并能够对其执行关系查询，那么应用
程序就可以使用最符合其需求的功能组合。
关系模型和文档模型的混合是未来数据库一条很好的路线。
. Codd对关系模型的原始描述实际上允许在关系模式中与JSON文档非常相似。他
称之为非简单域（nonsimple domains）。这个想法是，一行中的值不一定是一个像数
字或字符串一样的原始数据类型，也可以是一个嵌套的关系（表），因此可以把一个任
意嵌套的树结构作为一个值，这很像30年后添加到SQL中的JSON或XML支持

* 文档 schema
文档数据库有时称为无模式（schemaless），但这具有误导性，因为读取数据的代码通常假
定某种结构——即存在隐式模式，但不由数据库强制执行.
一个更精确的术语是读时
模式（schema-on-read）（数据的结构是隐含的，只有在数据被读取时才被解释），相应的
是写时模式（schema-on-write）（传统的关系数据库方法中，模式明确，且数据库确保所
有的数据都符合其模式）

* 为什么有些数据库修改数据很慢

大型表上运行 UPDATE 语句在任何数据库上都可能会很慢，因为每一行都需要重写。要是不可
接受的话，应用程序可以将 first_name 设置为默认值 NULL ，并在读取时再填充，就像使用
文档数据库一样。

* 查询局部性

查询的东西都在一个地方, 能一次性返回, 减少IO
* 一对多可以用树状结构表示

多对多关系是不同数据模型之间具有区别性的重要特征。如果你的应用程
序大多数的关系是一对多关系（树状结构化数据），或者大多数记录之间不存在关系，那么
使用文档模型是合适的。
### 编码
* json 

类似一颗多叉树

![image](https://user-images.githubusercontent.com/20329409/212016110-9b265c6e-5f60-4b6a-b9a5-8101b3607d0c.png)

json 和 xml , csv 等原生不能表达二进制数据, 均用 Unicode, 占用空间, 且均有一些不够兼容的小问题, 例如 xml 和 csv 无法区分数字和仅有数字代表的字符串, 
json 没有命名空间的概念, 相同的字段无法表达
### Thrift , Protocol Buffer
* 编码中如何压缩
int64 不一定占用八字节, 将每个字节的首位用于标识是否还有下一字节, 

* 如何向前兼容和向后兼容
向前兼容: 旧代码读新数据, 新数据中包含一个新的字段, 用一个新的 tag, 旧代码使用的是旧的 schema 不会识别出新的 tag , 即忽略新的字段
向后兼容: 新代码读取老数据, 新的 schema 的改动不能引起读取老数据的错误, 新的 schema 新加一个 tag (字段) ,旧的数据没有故读不到, 但是不能改变旧字段, 所以建议字段均是 optional(防止改成 required 校验失败), 删除字段意味着新代码不会读到旧代码的相关字段而已,
如果 tag 不变, 改变数据类型则需要考虑兼容, 否则会出现截断或数据丢失的风险. 

### Avro
TODO : avro 编码的图片

encoded bytes 并不包含 schema name 等, 也不包含指定 schema tag, 所以 writer schema 和 reader schema 必须要完全compatibility ; reader schema 通过 field name 找到 writer schema 对应的 field, 故通常来说不能改 field name, 但是可以在新的 schema 设置某一个field 为旧 schema 的 alias, 提供backward compatibility;

* 那么 writer schema 如何让 reader 知道呢? 
1. Large file with lots of records , include writer schema at the beginning of the file
2. Database with individually written records, match record with schema with schema version
3. Sending records over a network connection, 通过 avro RPC 框架, 在 conn 建立的时候指定 schema , 并用在整个 conn 的过程中
