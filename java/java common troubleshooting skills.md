---
在无数次troubleshooting 的痛苦经历中, 有一些经验和原则是必须被记下来的, 以便哪天头脑混乱的时候来查看, 以后每次找不出bug 的时候, 来这看看, 如果看完之后找出来了, 则 sillyCount++, 并更新时间; 

#### 心态篇
1. 程序和你的预期相违背时, 程序不会错乱, 错的是你的预期. 
2. 越急躁越难找出bug, 打字要快, 思维要缜密. 
3. 思路最重要, 当朝一个假设寻觅了半小时无结果时, 停下来整理思路五分钟, 一个小时的时候, 出去阳台走五分钟.
4. 一切线索都隐藏在代码中, 当网上资料混乱的时候, 静下心来看代码是最有效率的. 


#### 操作篇
1. 服务器上的问题, 当需要debug code 的时候, 可以使用arthas 和IDE remote debugger. 如果发现实际的代码调用和预期的不一致时, 例如应该进入Aclass.Bmethod 时, 却没有进入, 可以考虑是否是类冲突, 



> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTgwMDk2NTYxMV19
-->