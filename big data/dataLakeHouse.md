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
* file format

Hudi stores all ingested data in two different storage formats. The actual formats used are pluggable, but fundamentally require the following characteristics:
    - Scan-optimized columnar storage format (*ROFormat*). Default is [Apache Parquet](https://parquet.apache.org/)[.](https://avro.apache.org/)
    - Write-optimized row-based storage format (*WOFormat*). Default is [Apache Avro.](https://avro.apache.org/)
