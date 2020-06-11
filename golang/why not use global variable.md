* 背景
 今天领导看我没完全写好的代码, 说不要用全局变量, 然后两个人对原因都不太了解, 于是花了一个小时研究了一下

https://www.reddit.com/r/golang/comments/9i7ipd/passing_through_layers_vs_global_variables/

找到了三点原因: 
1. 依赖的问题, 用全局变量的话, 使用的那个 package 需要多一个 import
2. 封装,代码整洁的问题, 不用全局变量, 全部代码都封装在某一个 package 里面
3. 方便测试, mock 很简单, 相比于 interface 而言,  如果用 interface 接口, 



> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE1OTg1Nzk2NTldfQ==
-->