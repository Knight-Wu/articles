### 回退
checkout, revert, reset都可以执行回退工作

* checkout
git checkout branch_name // 用于切换已经存在的分支
git checkout commitId path // 用于将这些path下面的文件, head 指向commitId 提交后的状态, 中间的commit的历史不会被消除, 这是和reset的主要区别
git checkout HEAD path // 用户将path 下的文件恢复到HEAD 的状态,

* revert
* reset
git reset 


> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTExNDA3ODUzMDgsLTYzODIwNDE4NCwtMT
kwNzc4MF19
-->