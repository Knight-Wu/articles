* [6,824-schedule](https://pdos.csail.mit.edu/6.824/schedule.html)
*  MR [论文](https://pdos.csail.mit.edu/6.824/papers/mapreduce.pdf) 
## 问题
1. Inverted Index
2. We do not provide support for atomic two-phase com-mits of multiple output files produced by a single task.Therefore, tasks that produce multiple output files withcross-file consistency requirements should be determin-istic. 


# 重点
* MR 的不足或者不支持什么
1. 只支持函数式的编程, 不支持复杂的 pipeline 形式的, 相比于 spark. 

* 可以用 local execution 来 debug


# MR 实现逻辑
1. to get started, worker ask a task from master by RPC, and master return a filename
2. put the output of the X'th reduce task in the file mr-out-X
3. A mr-out-X file should contain one line per Reduce function output.
4.  If you change anything in the  mr/  directory, you will probably have to re-build any MapReduce plugins you use, with something like  go build -buildmode=plugin ../mrapps/wc.go

# 代码
由于这个课程要求不能在网络开放代码, 避免有人抄袭, 所以这里只放个人测试通过的repo 截图
![image](https://github.com/Knight-Wu/articles/assets/20329409/6cc778ab-b9ec-41e4-8e1d-11b9fe7b294a)
