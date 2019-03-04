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

* 树的分类
满二叉树, 完全二叉树, 平衡二叉树
![enter image description here](https://drive.google.com/uc?id=1WvdQnszfoBQkf94qiUDM2W_a4BWwArD8)
深度为k, 拥有2的k+1次方减一的节点的为满二叉树, 每一层的节点数都是最大节点数; 而若最后一层不是满的, 其余层都是满的, 则为完全二叉树, 具有n个节点的完全二叉树的深度为log2n +1(以2为底数)。深度为k的完全二叉树, 最少有2的k次方个节点, 最多有2的k+1 次方减一个节点. 

* AVL 树
平衡二叉搜索树（Self-balancing binary search tree）又被称为AVL树（有别于AVL算法），且具有以下性质：它是一 棵空树或它的左右两个子树的高度差的绝对值不超过1，并且左右两个子树都是一棵平衡二叉树。当只有一个根节点时, 根节点的高度定义为1.

平衡二叉树就是二叉树的构建过程中，每当插入一个结点，看是不是因为树的插入破坏了树的平衡性，若是，则找出最小不平衡树。在保持二叉树特性的前提下，调整最小不平衡子树中各个结点之间的链接关系，进行相应的旋转，使之成为新的平衡子树


**AVL**  **效率总结 :** 查找的时间复杂度维持在O(logN)，不会出现最差情况
AVL树在执行每个插入操作时最多需要1次旋转，其时间复杂度在O(logN)左右。
AVL树在执行删除时代价稍大，执行每个删除操作的时间复杂度需要O(2logN)。
* 红黑树
红黑树相对于AVL树来说，牺牲了部分平衡性以换取插入/删除操作时少量的旋转操作，整体来说性能要优于AVL树。

* AVL VS 红黑树
avl 更加平衡, 查找效率更高,但是插入和删除所需要的旋转代价也更高, 红黑树平均查找时间复杂度O(lgn), 最坏是


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
4. 叶子节点存有指向下个叶子节点的指针, 适合范围查询. 

* 为什么B+ 更适合做数据库和文件系统的数据类型呢
https://stackoverflow.com/questions/870218/differences-between-b-trees-and-b-trees
B+ 树的非叶子节点没有数据, 相同情况下比B 树在一个磁盘块中存储更多的节点, 需要的IO 次数就更低, 而且叶子节点有指向下个节点的指针, 不需要再从顶向下扫描周边叶子节点, 适合范围查询.


#### 排序

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
    if (start >= end)  
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
选一个基准数, 小于基准数的放到右边, 大于的放到左边, 并一直递归到数组只有一个元素, 再返回, 平均时间复杂度: O(nlgn), 

> 最好时间复杂度

数组已经按升序排好序, pivot 为中间元素, 则第一次遍历需要n 次比较, 然后均分成n/2 和 n/2 的两组进行递归, 满足O(n) = 2O(n/2)+ O(n), 则最好时间复杂度为O(nlgn)

> 最坏时间复杂度

数组元素为降序, 当需要排成升序的时候. 或者元素都相等的时候(没有使用三分法). 
详见 https://en.wikipedia.org/wiki/Quicksort  " Worst-case analysis"

> 版本一
```
static void QSRecu(int[] arr, int start, int end) {  
    if (start >= end)// 一定要注意这个返回, 不然就算start > end, 还是会一直进while 循环.  
  return;  
    int l = start;  
    int r = end;  
    int pivot = arr[(start+(end-start)/2) >> 1];  // 防止start+end overflow
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

> 版本二

```
// 思想: 把小于pivot 的数移到前面, 大于pivot 的数移到后面, 然后用一个指针指向第一个大于pivot 的数, 最后交换这个指针和末尾指针.
static void QSRecu1(int[] arr, int start, int end) {  
    if (start >= end)  
        return;  
    int l = start;  
    int pivot = arr[end];  
    for (int i = start; i < end; i++) {  
        if (arr[i] < pivot) {  
            if (i != l) {  
                swap(arr, i, l);  
            }  
            l++;  
        }  
    }  
    swap(arr, l, end);  
  
    QSRecu1(arr, start, l - 1);  
    QSRecu1(arr, l + 1, end);  
}
```

* 冒泡排序
每完全冒泡一次, 就把一个最大或最小的元素放到了数组的一端, 最坏情况需要O(n*n) 交换, 而插入排序最坏情况只需要O(n) 交换, 因为新插入的元素是可以跟有序子序列的每个元素进行比较, 而冒泡排序只能跟相邻元素进行比较
```
static void popS(int [] arr){  
     int len = arr.length;  
     for(int i=len-1;i>=1;i--){  
         for(int j=0;j+1<=i;j++){  
             if(arr[j] >arr[j+1]){  
                 int temp = arr[j];  
                 arr[j]=arr[j+1];  
                 arr[j+1]=temp;  
             }  
         }  
     }  
 }
```

* 插入排序
将新元素插入一个已经有序的子序列, 跟这个子序列的最后一个元素开始进行比较, 若小于则将子序列元素后移, 直到最后找到位置插入. 最坏情况只需要O(n) 次交换.

```
static void insertS(int[] arr) {  
    int len = arr.length - 1;  
    int i = 0;  
    int j = 0;  
    while (j + 1 <= len) {  
        i = j;  // j表示有序序列的末尾, i 表示需要插入的位置
        int temp = arr[j + 1];  
        while (i >= 0 && temp < arr[i]) {  
            arr[i + 1] = arr[i];  
            i--;  
        }  
        arr[i + 1] = temp;  // 注意需要前移一个位置插入
        j++;  // 注意子序列长度已经加一
    }  
}
```
* 选择排序
```
static void selectS(int[] arr) {  
    int lastI = arr.length - 1;  
    int j = 0;  
    int i;  
    while (j <= lastI) {// 外层排序表示要找最大值的次数, 为len-1 次  
  i = 0;  
        int maxI = i;  
  
        while (i <= lastI - j) {// 内层每次循环找到一个最大值, 并且放到最后,   
  if (arr[i] > arr[maxI]) {  
                maxI = i;  
            }  
            i++;  
        }  
        int temp = arr[maxI];  
        arr[maxI] = arr[lastI - j];  
        arr[lastI - j] = temp;  
        j++;// 表示找到了一个最大值, 并放到了最后  
  }  
}

```
* 堆排序
是最好的增量排序方式, 比二叉搜索树更加对cpu cache 友好, 
假设最后排序是升序, 首先将数组建立成一个最大堆, 最大的元素在arr[0], 次大的元素在其子节点, 然后将arr[0] 和arr[len-1]交换, 交换之后再以现在的根节点建堆, 将 arr[0]放在合适的位置. 
初始化建堆的时间复杂度O(n), 整个算法的时间复杂度O(nlgn)
```

static void heapS(int[] arr) {  
    int len = arr.length - 1;  
    for (int i = (len - 1) / 2; i >= 0; i--) {  
        buildHeap(arr, i, len);  
    }  
  
    for (int i = len; i > 0; i--) {  
  
        int temp = arr[0];  
        arr[0] = arr[i];  
        arr[i] = temp;  
        buildHeap(arr, 0, i - 1);  
    }  
}  
  
static void buildHeap(int[] arr, int i, int len) {  
    int l = i * 2 + 1;  
    int r = i * 2 + 2;  
    int moreI = l;  
    if (l > len) {  
        return;  
    }  
    if (r <= len && arr[r] > arr[moreI]) {  
        moreI = r;  
    }  
    if (arr[moreI] > arr[i]) {  
        int temp = arr[moreI];  
        arr[moreI] = arr[i];  
        arr[i] = temp;  
        buildHeap(arr, moreI, len);  
    }  
}

```

* 希尔排序
插入排序的变种, gap 由 len/2, len/4 递减, 直至变为gap=1的插入排序, 此时大部分元素已经排序好了.  
原始的算法实现在最坏的情况下需要进行O(n2)的比较和交换。V. Pratt的书[1]对算法进行了少量修改，可以使得性能提升至O(n log2 n)。这比最好的比较算法的O(n log n)要差一些。  

```
static void hillS(int[] arr) {  
    int len = arr.length - 1;  
    for (int gap = len >> 1; gap >= 1; gap = gap >> 1) {  
        for (int i = gap; i <= len; i++) {  
            int temp = arr[i];  
            int j = i - gap;  
            while (j >= 0 && temp < arr[j]) {  
                arr[j + gap] = arr[j];  
                j = j - gap;  
            }  
            arr[j + gap] = temp;  
        }  
  
    }  
  
}
```
* 桶排序
将元素放到几个子数组里面, 按照跟min 元素的差值来划分, 并在每个子数组中进行插入排序. 最坏时间复杂度O(n*n), 平均时间复杂度O(n), 见英文维基百科

* 二叉搜索树
插入是从顶向下, 递归的形式, 比节点小, 则比左儿子, 比节点大, 则比右儿子,  新插入的值总是落在叶子节点.

#### 资源
* 算法第四版
https://algs4.cs.princeton.edu/33balanced/
* coursera
https://www.coursera.org/learn/algorithms-part1/


### 查找
#### 布隆过滤器
用途在于数据量非常庞大的时候, 判断数据是否在集合中, 减少数据集的空间占用, 用一个bit 数组来保存, 假设有n个hash 函数, 新来的一个元素a, 
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

> 字符串

1. 如果字符串已经限定了是ASCII, 则可以用一个256 长度的int 数组来同等表示, 而且可以根据int 值的大小来排序, 排序的时间复杂度是O(n) .
2. 滑动窗口的思想

> 查找

1. 万变不离其中, 最快的查找就是二分查找, 时间复杂度O(lgn), 其次是线性时间复杂度.


> 二叉树

前序, 中序, 后序都是二叉树的深度优先搜索, 记住是深度, 二叉树是递归的结构, 最适合使用递归来解决.

```
public class TreeNode {  
    private TreeNode left;  
    private TreeNode right;  
    private int val;  
  
}  
  
// 二叉树层次遍历, 深度优先搜索  
public List<List<Integer>> levelDFS(TreeNode root) {  
    List<List<Integer>> list = new ArrayList<>();  
    dfsHelp(list, root, 0);  
    return list;  
}  
  
public void dfsHelp(List<List<Integer>> list, TreeNode root, int height) {  
    if (root == null) return;  
    if (height > list.size() - 1) {  
        list.add(new ArrayList<>());  
    }  
    List<Integer> list1 = list.get(height);  
    list1.add(root.val);  
    dfsHelp(list, root.left, height + 1);  
    dfsHelp(list, root.right, height + 1);  
}  
  
// 二叉树层次遍历, 广度优先搜索  
public List<List<Integer>> levelBFS(TreeNode root) {  
    List<List<Integer>> list = new ArrayList<>();  
    if (root == null) return list;  
    Queue<TreeNode> queue = new LinkedList<>();  
    queue.add(root);  
    while (!queue.isEmpty()) {  
        int levelNum = queue.size();  
        List<Integer> everyLevelList = new ArrayList<>(levelNum);  
        for (int i = 0; i < levelNum; i++) {  
            TreeNode node = queue.poll();  
            everyLevelList.add(node.val);  
            if (node.left != null) queue.add(node.left);  
            if (node.right != null) queue.add(node.right);  
        }  
        list.add(everyLevelList);  
  
    }  
    return list;  
}  
  
// 按照层次遍历入队列, 如果null 节点都出现在末尾则为完全二叉树  
public boolean ifCompleteTree(TreeNode root) {  
    if (root == null) return true;  
    Queue<TreeNode> queue = new LinkedList<>();  
    nodeToQueue(root, queue);  
    int a = Integer.MIN_VALUE;  
    int b = 0;  
    for (int i = 0; i < queue.size(); i++) {  
        if (queue.poll() == null) a = Math.min(a, i);  
        if (queue.poll() != null) b = i;  
    }  
    return a > b;  
  
}  
  
public void nodeToQueue(TreeNode root, Queue<TreeNode> queue) {  
    if (root != null) queue.add(root);  
    queue.add(root.left);  
    queue.add(root.right);  
}
```

> 数字题

因为所有的数字都是二进制, 可以用二进制中独有的运算, 位运算来达到四两拨千斤的效果, 

1. 位运算

* 负数如何用二进制表示

将绝对值用二进制表示, 然后取反, 然后加1, 最左边第一位为符号位, 若是负数则为1, 正数则为0; 
负数还原为正数: 第一位为符号位, 若为负数, 则先减一, 再取反. 

* 应用
 1. 异或, a^b, 不同的数异或结果为1 , 得出的二进制位哪位为1, 则哪位不同. 用于找只出现一次或两个只出现一次的数字 
 2. 高低位交换, 先左移, 再右移, 最后再或 


> 递归思想

递归可以想象成函数调用不断压栈, 直到满足**结束条件**, 最后一个调用首先返回. 
在递归调用之前的语句会正序执行(自顶向下), 再递归调用之后的语句会逆序执行(自下而上)

* 注意事项
1. 注意递归中, 传递的参数的类型, 基本类型例如int, char 等, 在函数栈的嵌套中拷贝一个新的值到下一个函数调用, 每个函数栈的变量的改变不影响其他函数栈; 但是如果是对象类型, 则传递的是对象的引用, 多个函数栈中会改变对象的值; 但是如果传递的是新的字符串, 则不会改变
2. 灵活使用递归中的全局变量和参数, 能让递归更易理解. 


#### 待解决问题
* paxos 的应用
* 回文字符串
https://leetcode.com/problems/rotate-string/solution/

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTExODQzMjI2MTAsLTExMTA0MDkzOTQsMj
EzNDgyNDg4MCwyOTU3NTAxMzcsMTAxMTQ1NDg4LC0xODQ5MjQx
MDE3LDMzNjU2OTA0Niw4MDM1ODA4MDYsLTE2NjM0MjAwMzMsLT
IwMzU0MDk5ODQsLTE2MzUxNjY5NDYsMTA2ODkwODU3LC0xNzI0
MDUyMjM2LDE2MzY0NjU1MDksLTE5MTQxMzY3OCwyMDUxNDg5MD
E2LC0xMzQwODMyOTc5LC05NDY0MzI3MzAsLTIxMzA1MDU0ODAs
OTE0MzI1NjM5XX0=
-->