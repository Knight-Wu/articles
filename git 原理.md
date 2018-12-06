### 回退
checkout, revert, reset都可以执行回退工作
[代码回滚：Reset、Checkout、Revert 的选择](https://github.com/geeeeeeeeek/git-recipes/wiki/5.2-%E4%BB%A3%E7%A0%81%E5%9B%9E%E6%BB%9A%EF%BC%9AReset%E3%80%81Checkout%E3%80%81Revert-%E7%9A%84%E9%80%89%E6%8B%A9)
* checkout
[checkout 检出之前的提交](https://github.com/geeeeeeeeek/git-recipes/wiki/2.5-%E6%A3%80%E5%87%BA%E4%B9%8B%E5%89%8D%E7%9A%84%E6%8F%90%E4%BA%A4)
git checkout branch_name // 用于切换已经存在的分支
git checkout commitId path // 用于将这些path下面的文件, head 指向commitId 提交后的状态, 中间的commit的历史不会被消除, 这是和reset的主要区别
git checkout HEAD path // 用户将path 下的文件恢复到HEAD 的状态
git checkout HEAD path/*   // 作用于 path 下面递归的所有的文件

这个命令就很好用了, 想看下某个目录之前某个提交后的状态, 但是只想影响本地仓库, 看完后, 再切回到HEAD 的状态, 例如想快速deploy 某个历史版本

但是有可能会导致detached HEAD, 
https://www.git-tower.com/learn/git/faq/detached-head-when-checkout-commit
问题: 到底s
```
$ git checkout -b test-branch 56a4e5c08 
...do your thing... 
$ git checkout master 
$ git branch -d test-branch

```
* reset
git reset 大致与git checkout 一致, 只是会清除掉历史记录, 并不能回到HEAD, 若要回退到HEAD, 则执行git pull

* revert
用于 public(公共)的修改, 与项目其他成员共享, 产生历史记录.
> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE3NjAxODA0OSwtODQ5ODI5NjY4LDEzOT
MxMDEyNDEsLTYzODIwNDE4NCwtMTkwNzc4MF19
-->