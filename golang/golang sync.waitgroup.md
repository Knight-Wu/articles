### sync.waitgroup

排查bug 的两点总结: 
1. waitgroup 要传指针
2. add 要在主线程做, 否则可能主线程都wait 了, 还没add , 就会出现和预期不符合的结果. 
>Typically this means the calls to Add should execute before the statement  creating the goroutine or other event to be waited for.

铭记一点: 越离奇, 越不符合之前认知的错误, 越复杂, 越要静下心来, 细细分析, 用最理性的思维, 判断, 很有可能需要推翻之前的很多认知, 这样才有可能越快找到问题. 

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTgyNzQzNTE1MCwtNDM5ODAxOTcwXX0=
-->