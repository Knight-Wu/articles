# dataLakeHouse vs data warehouse
<img width="1163" alt="image" src="https://user-images.githubusercontent.com/20329409/236136814-abc51193-8c40-446a-8854-7975eea3618e.png">

* only one storage on s3 or others , this storage is source of truth, not data duplicates on every storage for different purposes, 很难管理, 又贵.
* 存储计算分离
* table schema 支持 acid, schema change , 也更便宜相较之前 datawarehouse
* 不同层做不同事, 存储只管存, file format just for 如何更好的存一个文件的数据, table format: hudi 为了支持 acid , schema change 等表的功能, 管理很多文件. 再上层就是各种 service, 计算, 实时查询等.  
## old data warehouse
* pros and cons
<img width="699" alt="image" src="https://user-images.githubusercontent.com/20329409/236137238-07889866-cb58-4552-a0db-5fabfec1336d.png">

## dataLakeHouse 
* component

<img width="764" alt="image" src="https://user-images.githubusercontent.com/20329409/236137599-139f4f32-6700-48c9-b8c7-f51066cd3a7d.png">

* hudi vs iceberg vs datalake

hudi may is the best.
<img width="942" alt="image" src="https://user-images.githubusercontent.com/20329409/236150185-6d5a3b3b-4c13-493e-badd-19e61db7cb05.png">

## hudi 

### file layout

* Hudi organizes data tables into a directory structure under a base path on a distributed file system
* Tables are broken up into partitions
* Within each partition, files are organized into file groups, uniquely identified by a file ID
* Each file group contains several file slices
* Each slice contains a base file (.parquet) produced at a certain commit/compaction instant time, along with set of log files (.log.*) that contain inserts/updates to the base file since the base file was produced.
* Hudi adopts Multiversion Concurrency Control (MVCC), where compaction action merges logs and base files to produce new file slices and cleaning action gets rid of unused/older file slices to reclaim space on the file system.


![image](https://user-images.githubusercontent.com/20329409/236415238-217d2e5a-e478-4ef7-8a1c-652e292bc481.png)

### file format

Hudi stores all ingested data in two different storage formats. The actual formats used are pluggable, but fundamentally require the following characteristics:
    - Scan-optimized columnar storage format (*ROFormat*). Default is [Apache Parquet](https://parquet.apache.org/)[.](https://avro.apache.org/)
    - Write-optimized row-based storage format (*WOFormat*). Default is [Apache Avro.](https://avro.apache.org/)

### query type

* Snapshot Queries 
查最新的数据, 在 merge on read table 上类似准实时查询, 会在查询的时候做一些合并之后再查, 所以查询延迟高, 数据延迟低. 

* Incremental Queries
只查增量数据, 自从上一次 commit 或 compaction.

* read Optimized Queries
也是查最新的数据, 最新的快照, 但是只查列式的文件, 所以查询的就快. 但是有些数据还没有变成列式的之前就查不到, 所以数据延迟高, 查询延迟低. 


### copy on write table
![image](https://user-images.githubusercontent.com/20329409/236374615-bf97ebe5-43d3-4769-a5ac-077d55ee7932.png)

* update 的操作会在已有的 file group 新建一个文件. insert 新建一个 file group, 上图每个颜色是一个 file group.

* 会首先查 timeline 然后 filter file group 里面的一些文件. SQL queries running against such a table (eg: select count(*) counting the total records in that partition), first checks the timeline for the latest commit and filters all but latest file slices of each file group. 


### merge on read table
![image](https://user-images.githubusercontent.com/20329409/236375966-dc4a4508-8aad-4b31-bc5d-2f4fd94ec038.png)


snapshot 读会先把 delta log file 和列式文件合并之后再读, 读延迟高但是数据延迟低; 而 read optimzed 读则会只读列式文件, 读延迟低, 但是数据延迟高, 有些数据在 delta log 中没有读到. 




