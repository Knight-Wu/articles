# hiveSQL 

### int 和 string 直接比较的坑
int 类型直接用 != 或 = 和string 比较的时候, 会返回null , 见例子3, 

* 例子
> select case when 123 != '' then '1' else '2' end; 
> 输出 2 

>select case when cast(123 as string) != '' then '1' else '2' end;
>输出 1

>select 123 != '' ;
>输出 null

> int 类型的col , 插入 ‘’ 空字符串, 会转成null , 

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTU1Nzg2MzI4Ml19
-->