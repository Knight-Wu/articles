* int(3) vs int(4)
跟storage size 并没有关系, 仍然需要4 bytes, 
但是若指定了ZERO_FILL, 则不足三位会在前面补0, 

   if the stored value has less digits than  _x_,  `ZEROFILL`  will prepend zeros.
    
    > **INT(5) ZEROFILL**  with the stored value of 32 will show  **00032**  
    > **INT(5)**  with the stored value of 32 will show  **32**  
    > **INT**  with the stored value of 32 will show  **32**
    
  if the stored value has more digits than  _x_, it will be shown as it is.
    
    > **INT(3) ZEROFILL**  with the stored value of 250000 will show  **250000**  
    > **INT(3)**  with the stored value of 250000 will show  **250000**  
    > **INT**  with the stored value of 250000 will show  **250000**

* varchar(3) vs varchar(4)
并不影响storage size, storage size 只跟存储的具体string 有关, 若是单字节编码, 需要用一个字节或两个字节来存储长度, max len <= 65535 bytes, , 但是 varchar(3) 和 varchar(4) 查询中使用的内存并不一样, 后者让查询引擎使用更多的内存, 所以需要合理计算 varchar(x), string 长度大于 x 会截断
[https://stackoverflow.com/questions/1151667/what-are-the-optimum-varchar-sizes-for-mysql](https://stackoverflow.com/questions/1151667/what-are-the-optimum-varchar-sizes-for-mysql)

* varchar vs char
[https://dba.stackexchange.com/questions/2640/what-is-the-performance-impact-of-using-char-vs-varchar-on-a-fixed-size-field/2643#2643](https://dba.stackexchange.com/questions/2640/what-is-the-performance-impact-of-using-char-vs-varchar-on-a-fixed-size-field/2643#2643)

1. char 是定长, 在其他情况相同的时候, char 用于做索引会快 20% 
> The book MySQL Database Design and Tuning performed something marvelous on a MyISAM table to prove this

2. varchar 是变长, 如果实际存储的不是定长string, 那么肯定会更省空间. 
3. ALTER TABLE tblname ROW_FORMAT=FIXED;
让varchar 表现得跟char 一样, index 速度会加快, 但是存储的size 会增加不少

* varchar vs text

[https://stackoverflow.com/questions/25300821/difference-between-varchar-and-text-in-mysql](https://stackoverflow.com/questions/25300821/difference-between-varchar-and-text-in-mysql)
两个都是变长的存储方式, 但是因为mysql max row size 是 65535, 所以当需要存储很大的string 的时候最好用text, 因为他存储的是string 的ref, 只需要9-12 bytes, 
Reasons to use  `TEXT`:

-   If you want to store a paragraph or more of text
-   If you don't need to index the column
-   If you have reached the row size limit for your table

Reasons to use  `VARCHAR`:

-   If you want to store a few words or a sentence
-   If you want to index the (entire) column
-   If you want to use the column with foreign-key constraints



### mysql COLLATE
对于mysql中那些字符类型的列，如`VARCHAR`，`CHAR`，`TEXT`类型的列，都需要有一个`COLLATE`类型来告知mysql如何对该列进行排序和比较。简而言之，**COLLATE会影响到ORDER BY语句的顺序，会影响到WHERE条件中大于小于号筛选出来的结果，会影响**`**DISTINCT**`**、**`**GROUP BY**`**、**`**HAVING**`**语句的查询结果**。另外，mysql建索引的时候，如果索引列是字符类型，也**会影响索引创建**，

这是mysql的一个遗留问题，mysql中的`utf8`最多只能支持3bytes长度的字符编码，对于一些需要占据4bytes的文字，mysql的`utf8`就不支持了，要使用`utf8mb4`才行。

很多`COLLATE`都带有`_ci`字样，这是Case Insensitive的缩写，即大小写无关，也就是说"A"和"a"在排序和比较的时候是一视同仁的。对于那些`_cs`后缀的`COLLATE`，则是Case Sensitive，即大小写敏感的。

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE5NjYyNDcwNDVdfQ==
-->