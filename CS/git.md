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
https://stackoverflow.com/questions/10228760/fix-a-git-detached-head
问题: 到底为什么会 detached 
```
$ git checkout -b test-branch 56a4e5?c08 
...do your thing... 
$ git checkout master 
$ git branch -d test-branch

```
* reset
git reset 大致与git checkout 一致, 只是会清除掉历史记录, 并不能回到HEAD, 若要回退到HEAD, 则执行git pull

* revert
用于 public(公共)的修改, 与项目其他成员共享, 产生历史记录.


#### git merge

* 如何刚刚checkout 一个分支, 并没有任何commit 的时候, 就进行merge
https://stackoverflow.com/questions/4657009/how-to-merge-all-files-manually-in-git/36563486


#### git diff
git diff 两个分支或者两个 tag , 并且也可以指定路径, 若不加路径就是全项目 diff, 
eg: git diff tag1 tag2 [path]
或者你本地在 tag1, git diff tag2 [path]

#### Changing an Older or Multiple Commits
If you need to change the message of an older or multiple commits, you can use an interactive git rebase to change one or more older commits.

The rebase command rewrites the commit history, and it is strongly discouraged to rebase commits that are already pushed to the remote Git repository .

Navigate to the repository containing the commit message you want to change.

Type git rebase -i HEAD~N, where N is the number of commits to perform a rebase on. For example, if you want to change the 4th and the 5th latest commits, you would type:

git rebase -i HEAD~5
The command will display the latest X commits in your default text editor :

pick 43f8707f9 fix: update dependency json5 to ^2.1.1
pick cea1fb88a fix: update dependency verdaccio to ^4.3.3
pick aa540c364 fix: update dependency webpack-dev-server to ^3.8.2
pick c5e078656 chore: update dependency flow-bin to ^0.109.0
pick 11ce0ab34 fix: Fix spelling.

Rebase 7e59e8ead..11ce0ab34 onto 7e59e8ead (5 commands)
Move to the lines of the commit message you want to change and replace pick with reword:

reword 43f8707f9 fix: update dependency json5 to ^2.1.1
reword cea1fb88a fix: update dependency verdaccio to ^4.3.3
pick aa540c364 fix: update dependency webpack-dev-server to ^3.8.2
pick c5e078656 chore: update dependency flow-bin to ^0.109.0
pick 11ce0ab34 fix: Fix spelling.

ebase 7e59e8ead..11ce0ab34 onto 7e59e8ead (5 commands)
Copy
Save the changes and close the editor.

For each chosen commit, a new text editor window will open. Change the commit message, save the file, and close the editor.

fix: update dependency json5 to ^2.1.1
Copy
Force push the changes to the remote repository:

git push --force <remoteName> <branchName>