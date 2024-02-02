
# 参考资料
* 阿里云这里有常见的业务系统的设计方案

https://help.aliyun.com/zh/tablestore/use-cases/scheme-analysis?spm=a2c4g.11186623.0.i5
# 性能指标
## 硬件的性能
![image](https://github.com/Knight-Wu/articles/assets/20329409/cde90358-ea69-4e0b-bf7b-3b0bedbe1884)
参考: https://www.cs.cornell.edu/projects/ladis2009/talks/dean-keynote-ladis2009.pdf
* 每年的延迟分布情况:https://colin-scott.github.io/personal_website/research/interactive_latency.html

## 框架的性能
![image](https://github.com/Knight-Wu/articles/assets/20329409/1d910feb-9563-4ee2-b1d8-95e97cfeb4ae)


# 系统设计


## 要点
* 系统设计面试题的时候, 最重要的就是搞清楚问题和需求, 不做任何假设, 并且直接在对话框进行写, 如果说不用写, 就可能不需要回答这么细, 但为了保证自己的思路, 在自己的记事本上写. 如果一开始有多个方案多个思想, 可以说有多个方案, 一个个说; 
* 说白了任何系统设计都是涉及到写, 存储, 读, 
* 分布式系统最重要的就是要能水平扩展, 否则找个 mysql 存了任何业务问题都能解决. 写, 存储, 读都要能水平扩展. 
## 写
常见的就是 LSM 思想, 写WAL 和内存, 写 WAL 是顺序写, 将随机写转变为顺序写和内存写

## 存储
考虑数据如何分区, 如何复制, 如何设计表, 如何保持数据一致性, 这个一致性根据业务常见和读的需求不同而不同

## 读
如何保证性能, 如何保证符合业务要求的数据一致性, 例如转账是必须两边都要看到同样的符合转账记录的数据. 
### 数据分区(parition)
就是把热点数据分开, 让读写都可以水平扩展, 通过把数据分散到多台机器, 分布式解决单机的瓶颈问题
* 如何思考具体的分区策略呢

就是看热点数据是哪种, 怎么设定 key, 指定什么策略才能把热点 key 分散. 

* 数据密集型系统笔记-数据分区

  https://github.com/Knight-Wu/articles/blob/master/books/data_intensive_system_book_notes/chapter6_%E5%88%86%E5%8C%BA.md
## 常用数据
![image](https://github.com/Knight-Wu/articles/assets/20329409/81b38dda-f6b2-49dd-b1dd-0e3684636c83)

## 设计一个分布式限流器.
分为中心化的和去中心化的

### 中心化
中心化的就是依赖一个分布式的数据库, 每个需要限流的业务分配一个唯一的 key, 根据这个 key 请求, 判断是否达到了限流的上限

* 数据分区

根据 key 做 hash 分区, 

* 如何保证各个实例承载的压力一致呢?

由于限流请求很简单, 每个实例所能承载的限流 TPS 是固定的, 那么注册这个 key 的时候会知道需要支持多少 qps, 如果多个 key 的 qps 超过这个实例所能承载的上限就换到其他实例

* 数据访问

直接访问内存的数据结构, 如令牌桶和漏桶, 区别就是令牌桶会根据 tps 定期新增令牌, 能应对徒增; 漏桶就是严格控制请求平均; 内存的数据结构定期同步到磁盘, 但是对于某个 key 来说, 数据只在一个实例有单点问题, 那么就需要副本形成一个 replica group, 做数据拷贝</br>

* 数据同步的协议, 访问不了集群的情况. 

一般是主从同步, 但是这样开销比较大, 如果有 n 个副本, 则某个限流请求要访问 n+1 个实例, 都访问成功之后, 才能保证数据是同步的, 在限流这个场景对数据的精确度要求没这么高, 因为限流的数值一般是会留有 30% 的余地, </br>
所以应该是主副本定期将数据同步磁盘的时候, 同步到另外一个副本, 能容忍一次故障, 在副本数据跟主副本有延迟的情况下, 选择保守的策略, 偏向于将限流值调低, </br>
如果集群 down 或者多个实例挂掉的情况, 客户端应该能 failover 使用本地的限流器进行粗粒度的单机限流, 例如有一个转发层, 可以定期把key 对应的内存中限流的数据结构返回给客户端, 对于某个客户端来说只保留自己的 key 的限流数据, 在集群访问不了的情况下进行 failback

## 设计一个推特(feed 流系统)
### 详细设计
数据估算: 10,0000,0000 注册 user, 活跃用户 421 million monthly active users in 2023 and about 220 million daily actives
write : 600 tweet /s, 
read: 60 0000 tweet /s, 
tweet size: 280 character, 140 Chinese word, assume 0.2 kb


assume pull 5 tweets per fresh

#### 架构图
![image](https://github.com/Knight-Wu/articles/assets/20329409/8b1e36ce-7a32-4924-a477-392081b58131)

#### post api
 
post api, post with userId, content from client -> validation service -> storage service 
stoage service : 
1. gen tweet id 
2. 查 followRelation 获取被关注者 id list, 然后写 tweetId kv, append , 如果是大 v 则不写 tweetID kv
3. 写 userId kv
4. 写 tweetMeta kv

5. send to content to oss , save cost








 #### pull api

login serivce, validate -> pull api, query with timestamp, userId from client -> query service

query service:


1. query tweetId kv db, key is userId, val is 被关注者发帖的帖子 id list, 按照时间戳倒序排, 每次被关注者发帖的时候都会更新这个列表, 除了大 v, 如果关注者里面有大 v, 需要专门根据大 v 的 userID 查 tweetMeta kv, </br>
如果没有大 v, 就走 3, 实际上是 push 模式, 但是如果关注者很多几千个, 解决不了一下归并几千个 帖子 id lsit 的问题, 不知道哪些可以跳过, 不知道他们时间分布的情况, 而且读远多于写, 优化读取路径会性价比更高





2. query tweetMeta kv db, userid -> 自己发帖的 tweet id(including timestamp, meta) list, sorted, 每个 list 取五条, return query service a merge sorted list
2.1 写入的时候先写 cache, 再写 db, 查询也是先查 cache, 除非 cache 查不到,会造成部分用户看到最新帖子部分看不到 , 差异在秒级. 某个热点的 userId 集中在 redis 的某个机器, 可以添加读副本, 并且在上一层逻辑层的服务器内存加一层缓存, 因为应用服务器那层对某个 userId 的查询是分散在多台机器的, 
2.2 支持一直刷新, 类似深翻页:
 query the last tweet id list from lastedReturnId, 存储结构支持类似 skip list , index: range index, 因为 tweet id 肯定有序, 一个 chunk 指定开始 id 和结束 id, 就可以马上定位到哪个 chunk 包含需要的 id, 

2.3 如果大 v 的 userId 存在热点问题, 可以通过在 queryService 在应用服务器内存加缓存, 或者 tweetMeta kv 增加读的副本数解决吗, 如何感知某个 key 分布在多个 data partition 呢


3. query service 查询 tweet content by id, 
3.1 content cache 在 oss 前面,  tweetId is key, val is content, redis, 异步更新, 网红的 tweetId 会成为热点需要有多个副本都可以读取的能力. 

3.1 与 2.1 要先更新 3.1 保证能根据 id 找到 content, 如果实在找不到 content, 再去 query service 拿一个新的 id 会增加 RTT, 
所以可以查 10 个 content, 如果最新的五条能返回就直接返回, 如果某一条找不到 content, 就返回第六新的 content. 


4. get the 帖子内容从 oss, 并发的, 只需要等最新的五条, return to client

#### 写流程
1. 生成帖子 id, LSM 模式写入, 分区策略
#### 读流程
1. 根据用户 id 查询帖子, 然后 merge 排序展示. 
### 系统分类

Feed流种类较多，可根据接收的数据和关系数据进行分类。

根据接收数据的展示排序方式可以分为：
</br>
基于时间的Timeline：按发布的时间顺序排序，最后发布的排列在最上方，类似于朋友圈、微博，是最常见的Feed流形式。
</br>
基于推荐的Rank：按非时间的因子排序，例如按照用户的喜好度排序，选择出用户最想看的Top N结果排在上方。应用场景包括视频推荐、资讯推荐、商品推荐等。
</br>
根据用户数据中的关注关系可以分为：
</br>
单向关系系统：用户之间的关系有向，用户可以单方面关注另一个用户。因此系统中可能会存在大V，即有很多粉丝的用户。主要产品有微博、抖音等。
</br>
双向关系系统：用户之间的关系无向，用户建立关系后成为好友，即可互相交流，因此系统中不存在大V和粉丝的概念。用户使用此类产品时，会对时效性有更高的要求。主要产品有朋友圈、私信等。


### pull 模式, 读扩散
![image](https://github.com/Knight-Wu/articles/assets/20329409/0e492141-50c1-42a5-a60d-18776e9f25dc)
每一个内容发布者都有一个自己的发件箱（“我发布的内容”），每当我们发出一个新帖子，都存入自己的发件箱中。当我们的粉丝来阅读时，系统首先需要拿到粉丝关注的所有人，然后遍历所有发布者的发件箱，取出他们所发布的帖子，然后依据发布时间排序，展示给阅读者。
</br>
这种设计，阅读者读一次Feed流，后台会扩散为N次读操作（N等于关注的人数）以及一次聚合操作，因此称为读扩散。每次读Feed流相当于去关注者的收件箱主动拉取帖子，因此也得名拉模式。
</br>
这种模式的好处是底层存储简单，没有空间浪费。坏处是每次读操作会非常重，操作非常多。设想一下如果我关注的人数非常多，遍历一遍我所关注的所有人，并且再聚合一下，这个系统开销会非常大，时延上可能达到无法忍受的地步。
</br>因此读扩散主要适用系统中阅读者关注的人没那么多，并且刷Feed流并不频繁的场景。

### push 模式, 写扩散
![image](https://github.com/Knight-Wu/articles/assets/20329409/8d82cbc3-3417-461b-96cf-537da025781d)

据统计，大多数Feed流产品的读写比大概在100:1，也就是说大部分情况都是刷Feed流看别人发的朋友圈和微博，只有很少情况是自己亲自发一条朋友圈或微博给别人看。
</br>因此，读扩散那种很重的读逻辑并不适合大多数场景。我们宁愿让发帖的过程复杂一些，也不愿影响用户读Feed流的体验，因此稍微改造一下前面方案就有了写扩散
### pull 和 push 模式结合
![image](https://github.com/Knight-Wu/articles/assets/20329409/533671e5-e0d3-4596-917f-d79a0ae65f5b)

当何炅这种粉丝量超大的人发帖时，将帖子写入何炅的发件箱，另外提取出来何炅粉丝当中比较活跃的那一批（这已经可以筛掉大部分了），将何炅的帖子写入他们的收件箱。
</br>当一个粉丝量很小的路人甲发帖时，采用写扩散方式，遍历他的所有粉丝并将帖子写入粉丝收件箱。

对于那些活跃用户登录刷Feed流时，他直接从自己的收件箱读取帖子即可，保证了活跃用户的体验。
</br>
当一个非活跃的用户突然登录刷Feed流时，我们一方面需要读他的收件箱，另一方面需要遍历他所关注的大V用户的发件箱提取帖子，并且做一下聚合展示。
</br>在展示完后，系统还需要有个任务来判断是否有必要将该用户升级为活跃用户。因为有读扩散的场景存在，
</br>因此即使是混合模式，每个阅读者所能关注的人数也要设置上限，例如新浪微博限制每个账号最多可以关注2000人。
</br>如果不设上限，设想一下有一位用户把微博所有账号全部关注了，那他打开关注列表会读取到微博全站所有帖子，一旦出现读扩散，系统必然崩溃；即使是写扩散，他的收件箱也无法容纳这么多的微博。

读写混合模式下，系统需要做两个判断。一个是哪些用户属于大V，我们可以将粉丝量作为一个判断指标。另一个是哪些用户属于活跃粉丝，这个判断标准可以是最近一次登录时间等。

#### 如果使用大V/普通用户的切分，架构存在一定风险
例如某个大V突然发了一个很有话题性的Feed，那么就有可能导致整个Feed产品中的所有用户都没法读取新内容。以粉丝读取流程为例：

大V发送Feed消息。

大V使用拉模式。
</br>
大V的活跃粉丝（用户群A）开始通过拉模式读取大V的新Feed。有可能这部分用户可以在收件箱直接读取大 v 信息(提前写入), 毕竟是活跃用户
</br>
Feed内容太有话题性，快速传播。
</br>
未登录的大V粉丝（用户群B）开始登录产品，登录进去后自动刷新，再次通过拉模式读取大V的发件箱。
</br>
非粉丝（用户群C）去大V的个人页Timeline里面去围观，再次需要读取大V个人的Timeline，同读发件箱。
</br>
结果就是，平时正常流量只有用户群A，结果现在却是用户群A + 用户群B + 用户群C，流量增加了好几倍，甚至几十倍，导致读3路径的服务模块被打到server busy或者机器资源被打满，导致读取大V的读3路径无法返回请求。如果Feed产品中的用户都有关注大V，那么基本上所有用户都会卡死在读取大V的读3路径上，然后就没法刷新了。

##### 解决思路
  * 单个模块的不可用，不应该阻止整个关键的读Feed流路径。如果大V的无法读取，但是普通用户的要能返回，等服务恢复后，再补齐大V的内容即可。
  * 限流: 当模块无法承受这么大流量的时候，模块不应该完全不可服务，而应该能继续提供最大的服务能力，超过的拒绝掉。
###### 一个将用户帖子分散开, 提升读写的并发能力
因为大 v 采用 pull 模式, 可能存在某个大 v 的数据集中在某台机器造成热点, 所以让某个用户的帖子存在于多个 parition , 可以水平扩展, 增加读写并发度</br>
帖子 id = 用户 id + seqId, can get which data partitions to store those tweet when specify userId, 例如先通过 user id 做 hash 然后得到 prefix, 
</br> 每个 partition id 由 prefix 和 suffix 构成, 那么可以根据 prefix 定位这些 parition 在哪, 然后剩下的在某个 parition 中根据 userId 做扫描, 就是单机数据库的问题了
</br>
假设某个大 v 有 n 个 partition, 然后根据seqId 取模写入, 某个大 v 的 partition 应该不需要很多, 因为发帖很少, 总集群的 partition 数量根据所需的写入能力而定
</br> 一般都是写入数据写入 log 多副本之后, 就可以返回成功了, log 多副本因为顺序写磁盘加上写内存, 顺序写磁盘不会比内存慢多少, 就是一个 LSM 的架构. 
</br>
读取的时候每个 partition 读前5 条最新的, 假设第一页是五条, 最后在 merge, 因为每个 parition 的读取是并行的, 可以水平扩展, 读取时间取决于最慢的那个partition, 最后再按时间 merge, 而且大 v 的数量有限, 按大 v 的热度排序, 取最热的一批大 v, 把最新五条帖子写入缓存, 直到用完缓存, 
### 读写 feed 流程
#### 发布Feed流程
当你发布一条Feed消息的时候，流程是这样的：
1. Feed消息先进入一个队列服务。
2. 先从关注列表中读取到自己的粉丝列表，以及判断自己是否是大V。
3. 将自己的Feed消息写入个人页Timeline（发件箱）。如果是大V，写入流程到此就结束了。还可以将大 V 的帖子写入活跃粉丝的收件箱, 降低大 v 发热门贴时的读取压力. 
4. 如果是普通用户，还需要将自己的Feed消息写给自己的粉丝，如果有100个粉丝，那么就要写给100个用户，包括Feed内容和Feed ID。
5. 第三步和第四步可以合并在一起
6. 发布Feed的流程到此结束。

#### 读取Feed流流程
当刷新自己的Feed流的时候，流程是这样的：
1. 先去读取自己关注的大V列表
2. 去读取自己的收件箱，只需要一个GetRange读取一个范围即可，范围起始位置是上次读取到的最新Feed的ID，结束位置可以使当前时间，也可以是MAX，建议是MAX值。
3. 如果有关注的大V，则再次并发读取每一个大V的发件箱，如果关注了10个大V，那么则需要10次访问。
4. 合并2和3步的结果，然后按时间排序，返回给用户。


### 动态列表分页问题
传统的前端分页参数使用page_size和page_num，分表表示每页几条，以及当前是第几页。对于一个动态列表会有如下问题：


在T1时刻读取了第一页，T2时刻有人新发表了“内容11”，在T3时刻如果来拉取第二页，会导致错位出现，“内容6”在第一页和第二页都被返回了。事实上，但凡两页之间出现内容的添加或删除，都会导致错位问题。

为了解决这一问题，通常Feed流的分页入参不会使用page_size和page_num，而是使用last_id来记录上一页最后一条内容的id。前端读取下一页的时候，必须将last_id作为入参，后台直接找到last_id对应数据，再往后偏移page_size条数据，返回给前端，这样就避免了错位问题。

### 如何做sharding
例如推特表，存所有推文。userid 分表，某个网红就是热点；时间分表，读写最近是热点；比较好的是 userId+发帖时间戳 分表，最近一天的数据作为热数据分shard，比如100 个shard，然后hash 到不同shard，如果是hash 就要读多份，如果是range 就是有热点，为了读的体验，只能是range 写多份，读只读多数副本已经提交了的，类似kafka 的water mark。

### 优先级队列合并多个排序列表
```
// 用链表连接每个用户发的推特, 获取关注列表的所有推特, 就可以用优先级队列合并多个有序列表
public List<Integer> getNewsFeed(int userId) {
    List<Integer> res = new ArrayList<>();
    if (!userMap.containsKey(userId)) return res;
    // 关注列表的用户 Id
    Set<Integer> users = userMap.get(userId).followed;
    // 自动通过 time 属性从大到小排序，容量为 users 的大小
    PriorityQueue<Tweet> pq = 
        new PriorityQueue<>(users.size(), (a, b)->(b.time - a.time));

    // 先将所有链表头节点插入优先级队列
    for (int id : users) {
        Tweet twt = userMap.get(id).head;
        if (twt == null) continue;
        pq.add(twt);
    }

    while (!pq.isEmpty()) {
        // 最多返回 10 条就够了
        if (res.size() == 10) break;
        // 弹出 time 值最大的（最近发表的）
        Tweet twt = pq.poll();
        res.add(twt.id);
        // 将下一篇 Tweet 插入进行排序
        if (twt.next != null) 
            pq.add(twt.next);
    }
    return res;
}

class Tweet {
    private int id;
    private int time;
    private Tweet next;

    // 需要传入推文内容（id）和发文时间
    public Tweet(int id, int time) {
        this.id = id;
        this.time = time;
        this.next = null;
    }
}


class User {
    private int id;
    public Set<Integer> followed;
    // 用户发表的推文链表头结点
    public Tweet head;

    public User(int userId) {
        followed = new HashSet<>();
        this.id = userId;
        this.head = null;
        // 关注一下自己
        follow(id);
    }

    public void follow(int userId) {
        followed.add(userId);
    }

    public void unfollow(int userId) {
        // 不可以取关自己
        if (userId != this.id)
            followed.remove(userId);
    }

    public void post(int tweetId) {
        Tweet twt = new Tweet(tweetId, timestamp);
        timestamp++;
        // 将新建的推文插入链表头
        // 越靠前的推文 time 值越大
        twt.next = head;
        head = twt;
    }
}

```
## 求一个在线系统的某个时间段的最大在线人数和某个时刻的在线人数

给出的条件是一个日志数组 loga ，每个数组元素有三个属性，一个是用户id，一个是登陆时间 s，一个是登出时间 e。
不需要排序，用空间换时间，用一个时间数组 ta ，假设一天 x 秒，那么就是一个x 个元素的数组，遍历日志数组，
for l : loga{
    ta[s]++;
    ta[e]--;
}
某个时间点tx 的在线人数就是sum(ta[t], t从0 到tx),
某个时间段 t1 - t2的最大在线人数，
for t=t1;t <= t2; t++{
    用第一个式子求出ta[t1] 之后，再依次累加ta[t] , 并保留最大值即可.
}

## 秒杀系统, 抢红包系统
微信红包分为发, 抢, 拆, 看到领取结果几个阶段, 
发就是发一条特殊类型的消息到群聊里, 
然后抢可以设计为有个队列, 按照用户请求的顺序排队, 然后进队列前和后判断红包有没有抢完, 如果还有红包就到拆的页面, 就是多个请求并发 CAS 获取红包金额, 减红包数量, 红包的随机金额是根据发红包的时候
就计算出来的, 提前计算好, 有最大和最小值, 就是个随机序列, 然后CAS 成功就加一条红包记录, 就返回用户成功, 至于到账就是异步的, 可以在页面提醒高峰时到账慢或者只是简单加当前设备的余额

* 如何避免请求过多呢

因为先进队列, 后续并发CAS 是可以控制并发的, 然后如果红包没了, 就加一个缓存, 用红包id 做key, 后续的访问都走了缓存, 或者再客户端加缓存, 抢完后请求都不会发到服务端. 
## 微信朋友圈一篇文章几个人读过

采用发消息, 数据只显示在用户客户端上, 换了个客户端需要从其他客户端同步消息, 一个人读了一篇文章, 就给他所有好友发一条消息, 只累加文章读取数. 

## LRUCache
```
// here is LRUCache for only update when access(get)
public class LRUCache<K, V> {
  private LinkedHashMap<K, V> cache;
  private int cap;

  public LRUCache(int cap) {
    this.cap = cap;
    this.cache =
        new LinkedHashMap<>(cap, 0.75f, true) {
          @Override
          public boolean removeEldestEntry(Map.Entry<K, V> eldest) {
            return this.size() > cap;
          }
        };
  }

  public V get(K key) {
    // will unlink this node, and link it the the tail of the double linkedlist
    return cache.get(key);
  }

  public void put(K key, V val) {
    // will set the element to the tail, also make it to be the latest
    cache.put(key, val);
  }

  public static void main(String[] args) {
    LRUCache<Integer, Integer> ca = new LRUCache<>(2);
    ca.put(1, 1);
    ca.put(2, 2);
    ca.put(3, 3);
    Assert.assertNull(ca.get(1));
    Assert.assertEquals(2, ca.get(2).intValue());
    ca.put(4, 4);
    Assert.assertNull(ca.get(3));
  }
}
```
