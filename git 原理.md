### 回退
checkout, revert, reset都可以执行回退工作

* checkout
[checkout 检出之前的提交](https://github.com/geeeeeeeeek/git-recipes/wiki/2.5-%E6%A3%80%E5%87%BA%E4%B9%8B%E5%89%8D%E7%9A%84%E6%8F%90%E4%BA%A4)
git checkout branch_name // 用于切换已经存在的分支
git checkout commitId path // 用于将这些path下面的文件, head 指向commitId 提交后的状态, 中间的commit的历史不会被消除, 这是和reset的主要区别
git checkout HEAD path // 用户将path 下的文件恢复到HEAD 的状态
git checkout HEAD path/*   // 作用于 path 下面suo

这个命令就很好用了, 想看下某个目录之前某个提交后的状态, 但是只想影响本地仓库, 看完后, 再切回到HEAD 的状态, 例如想快速deploy 某个历史版本
* revert
* reset
git reset 


> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTg4ODAwOTAzOCwxMzkzMTAxMjQxLC02Mz
gyMDQxODQsLTE5MDc3ODBdfQ==
-->