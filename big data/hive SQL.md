# hiveSQL 

### int 和 string 直接比较的坑
* 例子
> select case when 123 != '' then '1' else '2' end; 
> 输出 2 

int 类型的col , 插入 ‘’ 空字符串, 会转成null 
> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTU0Mjc5NjY2MV19
-->