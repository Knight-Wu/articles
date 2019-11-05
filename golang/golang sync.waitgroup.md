### sync.waitgroup

排查bug 的两点总结: 
1. waitgroup 要传指针
2. add 要在主线程做, 否则可能主线程都wait 了, 还没add , 就会出现和预期不符合的结果. 
>Typically this means the calls to Add should execute before the statement  creating the goroutine or other event to be waited for.


> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTkyNDA1NjAxNiwtNDM5ODAxOTcwXX0=
-->