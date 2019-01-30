#### 时间复杂度

![enter image description here](https://drive.google.com/uc?id=1Z8qA1kFoTaSy9EXHAevCVd5Pk2nFHEVe)


![enter image description here](https://drive.google.com/uc?id=1ksKBcx4WHVwXCyum4H-JZCLX_Qy54Pj0)

* logN 底数是多少根本不重要
转自 [https://blog.csdn.net/bengxu/article/details/80320546](https://blog.csdn.net/bengxu/article/details/80320546)
> 假设有底数为2和3的两个对数函数，如上图。当X取N（数据规模）时，求所对应的时间复杂度得比值，即对数函数对应的y值，用来衡量对数底数对时间复杂度的影响。
比值为log2 N / log3 N，运用换底公式后得：(lnN/ln2) / (lnN/ln3) = ln3 / ln2，ln为自然对数，显然这三个常数，与变量N无关。
用文字表述：算法时间复杂度为log（n）时，不同底数对应的时间复杂度的倍数关系为常数，不会随着底数的不同而不同，因此可以将不同底数的对数函数所代表的时间复杂度，当作是同一类复杂度处理，即抽象成一类问题。
当然这里的底数2和3可以用a和b替代，a，b大于等于2，属于整数。a,b取值是如何确定的呢？
有点编程经验的都知道，分而治之的概念。排序算法中有一个叫做“归并排序”或者“合并排序”的算法，它用到的就是分而治之的思想，而它的时间复杂度就是N*logN，此算法采用的是二分法，所以可以认为对应的对数函数底数为2，也有可能是三分法，底数为3，以此类推。

### 树
* 树的遍历
最简单的划分：是深度优先（先访问子节点，再访问父节点，最后是第二个子节点）？还是广度优先（先访问第一个子节点，再访问第二个子节点，最后访问父节点）？ 深度优先可进一步按照根节点相对于左右子节点的访问先后来划分。如果把左节点和右节点的位置固定不动，那么根节点放在左节点的左边，称为前序（pre-order）、根节点放在左节点和右节点的中间，称为中序（in-order）、根节点放在右节点的右边，称为后序（post-order）。对广度优先而言，遍历没有前序中序后序之分：给定一组已排序的子节点，其“广度优先”的遍历只有一种唯一的结果。

* 中序遍历
因为左小右大, 可以根据中序遍历一颗二叉树来排序一个无序数组. 

* 红黑树作为自平衡树在jdk1.8 hashmap中 的作用

* B树
https://en.wikipedia.org/wiki/B-tree

According to Knuth's definition, a B-tree of order _m_ is a tree which satisfies the following properties:
1.  Every node has at most  _m_  children.
2.  Every non-leaf node (except root) has at least ⌈_m_/2⌉ child nodes.
3.  The root has at least two children if it is not a leaf node.
4.  A non-leaf node with  _k_  children contains  _k_  − 1 keys.
5.  All leaves appear in the same level.

![enter image description here](https://drive.google.com/uc?id=1EeNGv03evzr7zCnjHKEKAl_rgV0ebVCH)

B树相对于平衡二叉树的不同是，每个节点包含的关键字增多了，特别是在B树应用到数据库中的时候，数据库充分利用了磁盘块的原理（磁盘数据存储是采用块的形式存储的，每个块的大小为4K，每次IO进行数据读取时，同一个磁盘块的数据可以一次性读取出来）把节点大小限制和充分使用在磁盘快大小范围；把树的节点关键字增多后树的深度比原来的二叉树少了，减少数据查找的次数和复杂度;

* B+ 树
![enter image description here](https://drive.google.com/uc?id=1F_s695E7TcNkqRhdPszORZQt7sCdplTU)
1.  节点的子树数和关键字数相同（B 树是关键字数比子树数少一）
2.  节点的关键字表示的是子树中的最大数，在子树中同样含有这个数据(根节点的最大关键字其实就表示整个 B+ 树的最大元素。)
3.  叶子节点包含了全部数据，同时符合左小右大的顺序

* 为什么更适合做数据库和文件系统的数据类型呢
 In a **b-tree** you can store both keys and data in the internal and leaf nodes_ but in a **b+ tree** you have to store the data in the _leaf nodes only(由于中间节点不含数据)  所以一个磁盘块中可以含有更多的中间节点, IO 次数相对更低. 

#### 排序
* 手写堆排序, 建立堆的时间复杂度O(nlgn)

* 归并排序
分治的思想, 划分到两个子数组均只有一个元素, 再比较, 再merge 两个有序的子序列, 假设归并n 元素的时间复杂度是T(n) , 则T(n) = 2T(n/2)+O(n)(为合并两个元素个数为n/2 的有序子序列的时间复杂度)
, 经过推导得: T(n) =O(nlgn)

```
public static int[] mergeSort(int[] arr) {  
    int[] result = new int[arr.length];  
    int len = arr.length;  
    mergeSortRecu(arr, result, 0, len - 1);  
    return result;  
}  
  
public static void mergeSortRecu(int[] arr, int[] result, int start, int end) {  
    if (start < end)  
        return;// 递归返回, 此时start==end, 数组只有一个元素  
  int start1 = start;  
    int mid = (start + end) >> 1;  
    int end1 = mid;  
    int start2 = mid + 1;  
    int end2 = end;  
    mergeSortRecu(arr, result, start1, end1);  
    mergeSortRecu(arr, result, start2, end2);  
    int k = start;  
    while (start1 <= end1 && start2 <= end2) {  
        result[k++] = arr[start1] < arr[start2] ? arr[start1++] : arr[start2++];  
    }  
    while (start1 <= end1) {  
        result[k++] = arr[start1++];  
    }  
    while (start2 <= end2) {  
        result[k++] = arr[start2++];  
    }  
    for (int j = start; j <= end; j++) {// j<=end, 需要等于, 因为end 也是下标  
  arr[j] = result[j];// 拷贝到原数组, 之后原数组arr 就部分有序, 然后归并: 就是将两个有序子序列合并  
  }  
}

```

* 快速排序
选一个基准数, 小于基准数的放到右边, 大于的放到左边, 并一直递归到
```
static void QSRecu(int[] arr, int start, int end) {  
    if (start >= end)// 一定要注意这个返回, 不然就算start > end, 还是会一直进while 循环.  
  return;  
    int l = start;  
    int r = end;  
    int pivot = arr[(start + end) >> 1];  
    while (l <= r) {  
        while (arr[l] < pivot) {  
            l++;  
        }  
        while (arr[r] > pivot) {  
            r--;  
        }  
  
        if (l < r) {  
            int temp = arr[l];  
            arr[l] = arr[r];  
            arr[r] = temp;  
            l++;  
            r--;  
        } else if (l == r) {// 注意最后l和r 相等或相差一的两种情况.   
 l++;  
        }  
    }  
    QSRecu(arr, start, r);// l++之后, l是大于r 的  
  QSRecu(arr, l, end);  
}

```
#### 资源
* 算法第四版
https://algs4.cs.princeton.edu/33balanced/
* coursera
https://www.coursera.org/learn/algorithms-part1/


### 查找
#### 布隆过滤器
用途在于数据量非常庞大的时候, 减少数据集的空间占用, 用一个bit 数组来保存, 假设有n个hash 函数, 新来的一个元素a, 
插入: 将a 通过n 个hash 函数算出的结果分别对应bit 数组上的n 个bit 位, 分别置一; 
查询: 元素通过hash 函数算出的结果, 对应的bit 位上都是一, 则有可能在集合中, 因为n 个hash 都有可能冲突; 如果有一个bit 位为零, 则一定不存在集合中. 












#### 题目
> 排序

* 医院有M个医生，需要做的手术的手术时间用一个数组来描述，比如第2分钟、第5分钟、第8分钟等等，要求写一个函数，返回每一个手术由哪个医生做，要尽可能安排合理，即避免发生有的医生做很多手术有的医生一直在闲置的情况；

按照手术完成时间, 构建一个最小堆, 并用map 将医生id和手术完成时间关联起来; 碰到这种返回一个值的, 可以想最后返回的是哪个值, 比如这次是手术完成时间的最小值, 就可以往排序方面想. 



 * 给定一个数组, 每次调用的时候都随机输出所有的元素. 
 
public void solution( int [] arr){
  ran = new Random(seed);
  int a = ran.nextInt();
   int len = arr.length;
  for(int i=0;i<arr.length;i++){
     System.out.println(arr[a%len]);
    len--;
  }
}




#### 问题
* paxos 的应用
* 回文字符串
https://leetcode.com/problems/rotate-string/solution/
> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTIxMTU5MjYyNiwxNjE4MjMxNjYzLDYxNj
k2NzgwMiwzNDM1NzI0NzQsLTEwMjAwODU2NjgsMTgxNTM1ODQ0
OSwxNzUwMDM4ODMwLC02ODU2NzU0NjQsLTE0MzQwMzM4MDMsND
E4MTAzMDU5LDc4MTc1MTQyNSw0MTgxMDMwNTksLTE3NjAzNDI5
NiwtMTk1MDc3NDA5LDEwMjI5OTU4NTEsMTkxNDkyODg3Niw4MD
E4MTIzNzUsLTExMzA4NjA2MzMsMTYwMzM1NDQyMiwtMTMxNDIz
MTAzMl19
-->