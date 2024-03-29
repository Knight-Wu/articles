

# 待加强问题


# 常见技巧
* 当某些起始条件不满足, 例如数组第一个是n-1, n=0 时, 导致代码很复杂, 可以尝试把数组开辟成n+1, 总体加一, 往后移, n=1 存放n=0的值.
* 自顶向下, 例如nsum, 4 sum, 3 sum 等题目, 只要你能做出two sum, 然后把复杂情况拆解到简单情况, nsum 就能套用 2 sum, 3 sum 的做法, 递归和计算机编程的美妙就体现了. 详见:https://mp.weixin.qq.com/s/fSyJVvggxHq28a0SdmZm6Q
* 双指针, 链表中常见就是快慢指针, 数组中就是左右指针, 说白了就是两个变量, 减少循环次数
* 这些算法题一般都不会代码很长, 如果代码很长超过五十行, 或者if else 超过三个, 就要考虑是不是有些规律没想到, 导致没法归纳, 此时可以问下面试官, 问下提示或者是否可以归纳.
* 如果想高效地，等概率地随机获取元素，就要使用数组作为底层容器。然后取模.

* 如果要保持数组元素的紧凑性，可以把待删除元素换到最后，然后 pop 掉末尾的元素，这样时间复杂度就是 O(1) 了。因为最后一个元素remove 的时候, 置空即可, 如果是中间的元素remove, 就需要arraylist 的拷贝了, 然后我们需要额外的哈希表记录值到索引的映射, 然后把最后一个元素放到要remove 的元素的位置. 

# 双指针

```
给定一个长度为 n 的整数数组 height 。有 n 条垂线，第 i 条线的两个端点是 (i, 0) 和 (i, height[i]) 。

找出其中的两条线，使得它们与 x 轴共同构成的容器可以容纳最多的水。

返回容器可以储存的最大水量

      public int maxArea(int[] height) {
        int l = 0, r = height.length - 1;
        int ans = 0;
        while (l < r) {
            int area = Math.min(height[l], height[r]) * (r - l);
            ans = Math.max(ans, area);
            if (height[l] <= height[r]) {
                ++l;
            }
            else {
                --r;
            }
        }
        return ans;
    }
```

```
三数之和
给你一个整数数组 nums ，判断是否存在三元组 [nums[i], nums[j], nums[k]] 满足 i != j、i != k 且 j != k ，同时还满足 nums[i] + nums[j] + nums[k] == 0 。请

你返回所有和为 0 且不重复的三元组。

注意：答案中不可以包含重复的三元组。
和两数之和不同的是, 因为要去重, 所以要排序, 就不用map 了. 

   public List<List<Integer>> threeSum(int[] nums) {
      Arrays.sort(nums);
        List<List<Integer>> res = new LinkedList<>();
        
        for(int i = 0; i <= nums.length - 3; i++){
            if(nums[i] > 0){
                break;
            }
            int sum = 0 - nums[i];
            int l = i + 1;
            int r = nums.length - 1;
            while(l < r){
                int left = nums[l];
                int right = nums[r];
                int add = left + right;
                if(add < sum){
                    while(l < r && nums[l] == left){
                        l++;
                    }
                }else if(add > sum){
                    while(l < r && nums[r] == right){
                        r--;
                    }
                }else{
                    List<Integer> tmp = new LinkedList<>();
                    tmp.add(nums[i]);
                    tmp.add(left);
                    tmp.add(right);
                    res.add(tmp);
                     while(l < r && nums[l] == left){
                        l++;
                    }
                    while(l < r && nums[r] == right){
                        r--;
                    }
                }
            }
            while(i <= nums.length - 3 && nums[i] == nums[i+1]){
                i++;
            }
        }
        return res;
    }
```

```
两数之和
给定一个整数数组 nums 和一个整数目标值 target，请你在该数组中找出 和为目标值 target  的那 两个 整数，并返回它们的数组下标。

你可以假设每种输入只会对应一个答案。但是，数组中同一个元素在答案里不能重复出现。

你可以按任意顺序返回答案。

     public int[] twoSum(int[] nums, int target) {
        int[] indexs = new int[2];
        HashMap<Integer,Integer> hash = new HashMap<Integer,Integer>();
        for(int i = 0; i < nums.length; i++){
            if(hash.containsKey(nums[i])){
                indexs[0] = i;
                indexs[1] = hash.get(nums[i]);
                return indexs;
            }
            hash.put(target-nums[i],i);
        }
   
        return indexs;
    }
```

# flag 加减
```
给你一个 m 行 n 列的矩阵 matrix ，请按照 顺时针螺旋顺序 ，返回矩阵中的所有元素。
    public List<Integer> spiralOrder(int[][] matrix) {
        List<Integer> res = new LinkedList<>();
        int[][] m = matrix;
        int [][] step = new int [4][2];
        step[0] = new int [] {0,1};
        step[1] = new int [] {1, 0};
        step[2] = new int [] {0, -1};
        step[3] = new int [] {-1, 0};
        int index = 0;
        int row = matrix.length;
        int col = matrix[0].length;
        int sum = row * col;
        boolean [][] visit = new boolean [row][col];
        int r = 0;
        int c = 0;
        for(int i = 0; i < sum; i++){
            res.add(m[r][c]);
            visit[r][c] = true;
            int nextRow = r + step[index][0];
            int nextCol = c + step[index][1];
            if(nextRow < 0 || nextRow >= row || nextCol < 0 || nextCol >= col || visit[nextRow][nextCol]){
                index = (index + 1) % 4;
            }
            r = r + step[index][0];
            c = c + step[index][1];
        }
        return res;
    }
```
# 随机算法

## 构造任意 randN

```
下面将需要生成的随机数中最大值记为n

先去生成一个等概率产生0和1的方法：rand2() 。

实现：调用 rand7() ，2,4,6 返回0，1,3,5 返回1，7 重来。
将n转为二进制，看看是几位的二进制。

例如本题的 10 ，二进制为 1010 。也就是说仅仅需要4位二进制数就能表示10进制的10。 用这个 rand2() 方法生成n的每个二进制位。
创建一个数为 rand2() ，每次左移一位再加 rand2() 。

如果出现不符合题意的情况，再次调用 rand10() 。

例如如上步骤可能生成 0000 , 1011 、…… 、1111 （分别为0,11、 …… 、15），显然这些都是不在[1,10]中的正整数，所以只能重新生成。


    public int rand10() {
        int res = rand2();
        for(int i = 0; i < 3; i++){
            res = res << 1;
            res = rand2() | res;
        }
        if(res >= 1 && res <= 10){
            return res;
        }
        return rand10();
    }

    public int rand2(){
        int res = rand7();
        return res == 7 ? rand2() : res % 2;
    }

```
如何判断一个算法是不是随机均等的, 可以先算出随机的结果总数, 然后看是否能够产生这么多结果. 以及结果间是否是均等的.

* 如何映射一维数组到二维空间. 
```
class Game {
    int m, n;
    // 长度为 m * n 的一维棋盘
    // 值为 true 的地方代表有雷，false 代表没有雷
    boolean[] board;

    // 将二维数组中的坐标 (x, y) 转化为一维数组中的索引
    int encode(int x, int y) {
        return x * n + y;
    }

    // 将一维数组中的索引转化为二维数组中的坐标 (x, y)
    int[] decode(int index) {
        return new int[] {index / n, index % n};
    }
}
```

* 如何随机打乱一个数组?

分析洗牌算法正确性的准则：产生的结果必须有 n! 种可能。这个很好解释，因为一个长度为 n 的数组的全排列就有 n! 种，也就是说打乱结果总共有 n! 种。算法必须能够反映这个事实，才是正确的。

有了这个原则再看代码应该就容易理解了：

对于 nums[0]，我们把它随机换到了索引 [0, n) 上，共有 n 种可能性；

对于 nums[1]，我们把它随机换到了索引 [1, n) 上，共有 n - 1 种可能性；

对于 nums[2]，我们把它随机换到了索引 [2, n) 上，共有 n - 2 种可能性；

以此类推，该算法可以生成 n! 种可能的结果，所以这个算法是正确的，能够保证随机性。

```
// leetcode 384
class Solution {
    private int[] nums;
    private Random rand = new Random();
    
    public Solution(int[] nums) {
        this.nums = nums;
    }
    
    public int[] reset() {
        return nums;
    }
    
    // 洗牌算法
    public int[] shuffle() {
        int n = nums.length;
        int[] copy =  Arrays.copyOf(nums, n);
        for (int i = 0 ; i < n; i++) {
            // 生成一个 [i, n-1] 区间内的随机数
            int r = i + rand.nextInt(n - i);
            // 交换 nums[i] 和 nums[r]
            swap(copy, i, r);
        }
        return copy;
    }
    
    private void swap(int[] nums, int i, int j) {
        int temp = nums[i];
        nums[i] = nums[j];
        nums[j] = temp;
    }
}

```

* 给你一个未知长度的单链表，请你设计一个算法，只能遍历一次，随机地返回链表中的一个节点。

```
/* 返回链表中一个随机节点的值 */
int getRandom(ListNode head) {
    Random r = new Random();
    int i = 0, res = 0;
    ListNode p = head;
    // while 循环遍历链表
    while (p != null) {
        i++;
        // 生成一个 [0, i) 之间的整数
        // 这个整数等于 0 的概率就是 1/i
        if (0 == r.nextInt(i)) {
            res = p.val;
        }
        p = p.next;
    }
    return res;
}
```

我们来证明一下，假设总共有 n 个元素，我们要的随机性无非就是每个元素被选择的概率都是 1/n 对吧，那么对于第 i 个元素，它被选择的概率就是：

图片

第 i 个元素被选择的概率是 1/i，在第 i+1 次不被替换的概率是 1 - 1/(i+1)，在第 i+2 次不被替换的概率是 1 - 1/(i+2)，以此类推，相乘的结果是第 i 个元素最终被选中的概率，也就是 1/n。因此，该算法的逻辑是正确的。

同理，如果要在单链表中随机选择 k 个数，只要在第 i 个元素处以 k/i 的概率选择该元素，以 1 - k/i 的概率保持原有选择即可。代码如下：

```
/* 返回链表中 k 个随机节点的值 */
int[] getRandom(ListNode head, int k) {
    Random r = new Random();
    int[] res = new int[k];
    ListNode p = head;

    // 前 k 个元素先默认选上
    for (int i = 0; i < k && p != null; i++) {
        res[i] = p.val;
        p = p.next;
    }

    int i = k;
    // while 循环遍历链表
    while (p != null) {
        i++;
        // 生成一个 [0, i) 之间的整数
        int j = r.nextInt(i);
        // 这个整数小于 k 的概率就是 k/i
        if (j < k) {
            res[j] = p.val;
        }
        p = p.next;
    }
    return res;
}
```
图片

对于某个 i 来说, k 个位置中选中这个位置的概率是 k/i, 那么后续这个位置不被选中, 对于 i+1 来说, 选中这个位置的概率是A = k/(i+1) / k , 不选中就是 1 - A, 以此类推


类似的，回到扫雷游戏的随机初始化问题，我们可以写一个这样的 sample 抽样函数：


```
// 在区间 [lo, hi) 中随机抽取 k 个数字
int[] sample(int lo, int hi, int k) {
    Random r = new Random();
    int[] res = new int[k];

    // 前 k 个元素先默认选上
    for (int i = 0; i < k; i++) {
        res[i] = lo + i;
    }

    int i = k;
    // while 循环遍历数字区间
    while (i < hi - lo) {
        i++;
        // 生成一个 [0, i) 之间的整数
        int j = r.nextInt(i);
        // 这个整数小于 k 的概率就是 k/i
        if (j < k) {
            res[j] = lo + i - 1;
        }
    }
    return res;
}
```
# 前缀和数组

 前缀和主要适用的场景是原始数组不会被修改的情况下，频繁查询某个区间的累加和。

这个技巧在生活中运用也挺广泛的，比方说，你们班上有若干同学，每个同学有一个期末考试的成绩（满分 100 分），那么请你实现一个 API，输入任意一个分数段，返回有多少同学的成绩在这个分数段内。

那么，你可以先通过计数排序的方式计算每个分数具体有多少个同学，然后利用前缀和技巧来实现分数段查询的 API：
```
int[] scores; // 存储着所有同学的分数
// 试卷满分 100 分
int[] count = new int[100 + 1]
// 记录每个分数有几个同学
for (int score : scores)
    count[score]++
// 构造前缀和
for (int i = 1; i < count.length; i++)
    count[i] = count[i] + count[i-1];

// 利用 count 这个前缀和数组进行分数段查询
```
# 差分数组
差分数组的主要适用场景是频繁对原始数组的某个区间的元素进行增减。

比如说，我给你输入一个数组 nums，然后又要求给区间 nums[2..6] 全部加 1，再给 nums[3..9] 全部减 3，再给 nums[0..4] 全部加 2，再给…

一通操作猛如虎，然后问你，最后 nums 数组的值是什么？

常规的思路很容易，你让我给区间 nums[i..j] 加上 val，那我就一个 for 循环给它们都加上呗，还能咋样？这种思路的时间复杂度是 O(N)，由于这个场景下对 nums 的修改非常频繁，所以效率会很低下。

这里就需要差分数组的技巧，类似前缀和技巧构造的 prefix 数组，我们先对 nums 数组构造一个 diff 差分数组，diff[i] 就是 nums[i] 和 nums[i-1] 之差：
```
// 差分数组工具类
class Difference {
    // 差分数组
    private int[] diff;
    
    /* 输入一个初始数组，区间操作将在这个数组上进行 */
    public Difference(int[] nums) {
        assert nums.length > 0;
        diff = new int[nums.length];
        // 根据初始数组构造差分数组
        diff[0] = nums[0];
        for (int i = 1; i < nums.length; i++) {
            diff[i] = nums[i] - nums[i - 1];
        }
    }

    /* 给闭区间 [i, j] 增加 val（可以是负数）*/
    public void increment(int i, int j, int val) {
        diff[i] += val; // means ele in num[i] += val, so all eles whose index >= i += val 
        if (j + 1 < diff.length) {
            diff[j + 1] -= val; // means ele in num[j+1] -= val, so all eles whose index >= j+1 -= val 
        }
    }

    /* 返回结果数组 */
    public int[] result() {
        int[] res = new int[diff.length];
        // 根据差分数组构造结果数组
        res[0] = diff[0];
        for (int i = 1; i < diff.length; i++) {
            res[i] = res[i - 1] + diff[i];
        }
        return res;
    }
}
```

# 旋转矩阵

顺时针旋转九十度:
我们可以先将 n x n 矩阵 matrix 按照左上到右下的对角线进行镜像对称,
然后再对矩阵的每一行进行反转,
发现结果就是 matrix 顺时针旋转 90 度的结果.
```
// 将二维矩阵原地顺时针旋转 90 度
public void rotate(int[][] matrix) {
    int n = matrix.length;
    // 先沿对角线镜像对称二维矩阵
    for (int i = 0; i < n; i++) {
        for (int j = i; j < n; j++) {
            // swap(matrix[i][j], matrix[j][i]);
            int temp = matrix[i][j];
            matrix[i][j] = matrix[j][i];
            matrix[j][i] = temp;
        }
    }
    // 然后反转二维矩阵的每一行
    for (int[] row : matrix) {
        reverse(row);
    }
}

// 反转一维数组
void reverse(int[] arr) {
    int i = 0, j = arr.length - 1;
    while (j > i) {
        // swap(arr[i], arr[j]);
        int temp = arr[i];
        arr[i] = arr[j];
        arr[j] = temp;
        i++;
        j--;
    }
}
```

仔细想想，旋转二维矩阵的难点在于将「行」变成「列」，将「列」变成「行」，而只有按照对角线的对称操作是可以轻松完成这一点的，对称操作之后就很容易发现规律了。
既然说道这里，我们可以发散一下，如何将矩阵逆时针旋转 90 度呢？
思路是类似的，只要通过另一条对角线镜像对称矩阵，然后再反转每一行，就得到了逆时针旋转矩阵的结果：

# 矩阵的螺旋遍历
解题的核心思路是按照右、下、左、上的顺序遍历数组，并使用四个变量圈定未遍历元素的边界：
```
List<Integer> spiralOrder(int[][] matrix) {
    int m = matrix.length, n = matrix[0].length;
    int upper_bound = 0, lower_bound = m - 1;
    int left_bound = 0, right_bound = n - 1;
    List<Integer> res = new LinkedList<>();
    // res.size() == m * n 则遍历完整个数组
    while (res.size() < m * n) {
        if (upper_bound <= lower_bound) {
            // 在顶部从左向右遍历
            for (int j = left_bound; j <= right_bound; j++) {
                res.add(matrix[upper_bound][j]);
            }
            // 上边界下移
            upper_bound++;
        }
        
        if (left_bound <= right_bound) {
            // 在右侧从上向下遍历
            for (int i = upper_bound; i <= lower_bound; i++) {
                res.add(matrix[i][right_bound]);
            }
            // 右边界左移
            right_bound--;
        }
        
        if (upper_bound <= lower_bound) {
            // 在底部从右向左遍历
            for (int j = right_bound; j >= left_bound; j--) {
                res.add(matrix[lower_bound][j]);
            }
            // 下边界上移
            lower_bound--;
        }
        
        if (left_bound <= right_bound) {
            // 在左侧从下向上遍历
            for (int i = lower_bound; i >= upper_bound; i--) {
                res.add(matrix[i][left_bound]);
            }
            // 左边界右移
            left_bound++;
        }
    }
    return res;
}

```

# 滑动窗口算法

回顾一下，遇到子数组/子串相关的问题，你只要能回答出来以下几个问题，就能运用滑动窗口算法：

1、什么时候应该扩大窗口？

2、什么时候应该缩小窗口？

3、什么时候应该更新答案？
```
/* 滑动窗口算法框架 */
void slidingWindow(string s) {
    unordered_map<char, int> window;
    
    int left = 0, right = 0;
    while (right < s.size()) {
        // c 是将移入窗口的字符
        char c = s[right];
        // 增大窗口
        right++;
        // 进行窗口内数据的一系列更新
        ...
        
        // 判断左侧窗口是否要收缩
        while (window needs shrink) {
            // d 是将移出窗口的字符
            char d = s[left];
            // 缩小窗口
            left++;
            // 进行窗口内数据的一系列更新
            ...
        }
    }
}
```
实战题目:
```
//https://leetcode.com/problems/minimum-window-substring/ , 注意Integer 需要 equal 方法判断相等.
 public String minWindow(String s, String t) {
       // 记录t 以及 滑动窗口window中 字符与个数的映射关系
        HashMap<Character, Integer> window_map = new HashMap<>();
        HashMap<Character, Integer> t_map = new HashMap<>();
        for (int i = 0; i < t.length(); i++) {
            char c1 = t.charAt(i);
            t_map.put(c1, t_map.getOrDefault(c1, 0) + 1);
        }
        // 双指针, 
        int left = 0;
        int right = 0;
        int count = 0;
        // 用于更新满足的窗口window的长度,如果是len一直是MAX_VALUE，说明没有满足的串
        int len = Integer.MAX_VALUE;
        // 用于记录window串的起始位置，则返回 s[start, len]
        int start = 0;

        while (right < s.length()) {
            char c = s.charAt(right);
            right++;
            // 只要c是t中出现的字符就更新
            if (t_map.containsKey(c)) {
                window_map.put(c, window_map.getOrDefault(c, 0) + 1);
                // 更新c字符出现的次数
                if (window_map.get(c).equals(t_map.get(c))) {
                    count++;
                }
            }
            
            // ----------------------------------
            // 收缩window的长度
            while (count == t_map.size()) {
                // 更新并记录window的长度,以及window的起始位置start
                if (right - left < len) {
                    start = left;
                    len = right - left;
                }

                char d = s.charAt(left);
                left++;

                if (t_map.containsKey(d)) {
                    if (window_map.get(d).equals(t_map.get(d))) {
                        count--;
                    }
                    window_map.put(d, window_map.get(d) - 1);
                }
            }
        } 
        return len == Integer.MAX_VALUE ? "" : s.substring(start, start + len);
    }
```
# 二分查找
* 什么时候可以想到用二分查找 

可以从题目中抽象出一个自变量 x, 一个关于x 的函数 f(x), 以及一个目标值 target, 同时要满足 f(x) 关于 x 单调递增或递减. 并且题目让你计算满足 f(x) == target 时 x 的值. 
例如 https://leetcode.cn/problems/koko-eating-bananas/, x 是吃香蕉的速度, f(x) 是吃完所有香蕉需要的时间. 显然单调递减. 那么吃香蕉的时间限制在 H 之内就是target 的值. 
找到 x 的取值范围作为二分搜索的区间 left 和right

```
// https://leetcode.com/problems/koko-eating-bananas

  public int minEatingSpeed(int[] piles, int h) {
        int right = 0;
        for(int n : piles){
            right += n;
        }
        int left = 1;
        
        right = 1000000000;
        long res = right;
        while(left <= right){
            int mid = left + (right - left) / 2;
            int v = mid;
            long time = 0 ;
            for(int n : piles){
                if(v < n){
                    if(n % v != 0){
                        time += 1;
                    }
                    time +=  (n / v);
                }else{
                    time += 1;
                }
            }
            if( time > h){
                left = mid + 1;
            }else if( time <= h){
                right = mid - 1;
                res = Math.min(res, v);
            }
        }
        return (int)res;
    }
```

```
// https://leetcode.com/problems/split-array-largest-sum, 这题可以抽象成, 码头有一堆货物, 要求最多用 k 天送完货物, 要求每天运送最少的货物量是多少. 例如 https://leetcode.com/problems/capacity-to-ship-packages-within-d-days/, 简直和这一题是一样的. 
  public int splitArray(int[] nums, int k) {
        int max = 0;
        int right = 0;
        for(int n : nums){
            max = Math.max(max, n);
            right += n;
        }
        int left = max;
        int res = right;
        while(left <= right){
            int mid = left + (right - left)/2;
            int day = fun(nums, mid);
            if(day <= k){ // 注意这个, 只要是小于等于 k 的, 都可以分割成 k 因为后面的元素可以单独一个元素就作为一个子数组. 
                res = Math.min(res, mid); 
                right = mid - 1;
            }else if(day > k){
                left = mid + 1;
            }
        }
        return res;
    }

    int fun(int [] nums, int x){
        int day = 0;
        int cap = x;
        for(int i = 0; i < nums.length; i++){
            if(cap >= nums[i]){
                cap -= nums[i];
            }else{
                day++;
                cap = x;
                i--;
            }
        }
        if(cap != x){
            day++;
        }
        return day;
    }
```
* 技巧

分析二分查找的一个技巧是：不要出现 else，而是把所有情况用 else if 写清楚，这样可以清楚地展现所有细节.
另外提前说明一下，计算 mid 时需要防止溢出，代码中 left + (right - left) / 2 就和 (left + right) / 2 的结果相同，但是有效防止了 left 和 right 太大，直接相加导致溢出的情况。

```
int binary_search(int[] nums, int target) {
    int left = 0, right = nums.length - 1; 
    while(left <= right) { // 表示查找的区间, left > right 时才无区分可找
        int mid = left + (right - left) / 2; // 用减法防止两者相加溢出
        if (nums[mid] < target) {
            left = mid + 1;
        } else if (nums[mid] > target) {
            right = mid - 1; 
        } else if(nums[mid] == target) {
            // 直接返回
            return mid;
        }
    }
    // 直接返回
    return -1;
}


// 找左侧边界, 例如: [1,2,2,2,3], target=2, 左侧边界是 1
int left_bound(int[] nums, int target) {
    int left = 0, right = nums.length - 1;
    while (left <= right) {
        int mid = left + (right - left) / 2;
        if (nums[mid] < target) {
            left = mid + 1;
        } else if (nums[mid] > target) {
            right = mid - 1;
        } else if (nums[mid] == target) {
            // 别返回，锁定左侧边界
            right = mid - 1;
        }
    }
    // 判断 target 是否存在于 nums 中
    // 此时 target 比所有数都大，返回 -1
    if (left == nums.length) return -1;
    // 判断一下 nums[left] 是不是 target
    return nums[left] == target ? left : -1;
}

// 找右侧边界, 例如: [1,2,2,2,3], target=2, 左侧边界是 1
int right_bound(int[] nums, int target) {
    int left = 0, right = nums.length - 1;
    while (left <= right) {
        int mid = left + (right - left) / 2;
        if (nums[mid] < target) {
            left = mid + 1;
        } else if (nums[mid] > target) {
            right = mid - 1;
        } else if (nums[mid] == target) {
            // 别返回，锁定右侧边界
            left = mid + 1; 
        }
    }
    // 此时 left - 1 索引越界
    if (left - 1 < 0) return -1; // 如果target 很小, 找不到, 那么 left 一直为 0, 要返回的是下标为 left -1 的数据, 会越界.
    // 判断一下 nums[left] 是不是 target
    return nums[left - 1] == target ? (left - 1) : -1; // 为什么是 left -1 因为 left = mid + 1, 最后都加了1, 
```



# 优先级队列, 最大堆或最小堆

按照元素的大小去出队列, 又称最大堆, 或最小堆, 当碰到求一批元素的最大或最小值时, 可以直接用.
最大堆, 实际就是一个二叉树, 根元素为最大, 大于或等于左节点和右节点, 子树也满足最大堆. 

```
   // 这样就是最小堆, 反过来 b-a 就是最大堆
    PriorityQueue<Integer> q = new PriorityQueue<>(size, (a,b) -> (a.val - b.val));
    int min = q.pop();
```

# 单调栈
用于解决找到下一个更大的数此类问题, 用途较窄, 指的是从栈顶到栈底是单调递增或者递减的栈, 可以理解成一个临时存放数据的地方, 要怎么去想是单调递增还是递减呢, 就看什么数据需要暂存在栈里, 也就是不能马上返回或者解决的, 这样就好想, 符合单调的数据就一直压栈, 否则就弹出再压栈. 
# 单调队列
```
https://leetcode.cn/problems/sliding-window-maximum/
239. 滑动窗口最大值
给你一个整数数组 nums，有一个大小为 k 的滑动窗口从数组的最左侧移动到数组的最右侧。你只可以看到在滑动窗口内的 k 个数字。滑动窗口每次只向右移动一位。

返回 滑动窗口中的最大值 。

输入：nums = [1,3,-1,-3,5,3,6,7], k = 3
输出：[3,3,5,5,6,7]
解释：
滑动窗口的位置                最大值
---------------               -----
[1  3  -1] -3  5  3  6  7       3
 1 [3  -1  -3] 5  3  6  7       3
 1  3 [-1  -3  5] 3  6  7       5
 1  3  -1 [-3  5  3] 6  7       5
 1  3  -1  -3 [5  3  6] 7       6
 1  3  -1  -3  5 [3  6  7]      7


    public int[] maxSlidingWindow(int[] nums, int k) {
        Deque<Integer> q = new LinkedList<>();
        int [] res = new int [nums.length - k + 1];
        for(int i = 0; i < k; i++){
            while(!q.isEmpty() && nums[i] >= nums[q.peekLast()]){
                q.pollLast();
            }
            q.offerLast(i);
        }
        res[0] = nums[q.peekFirst()];
        for(int i = k; i < nums.length; i++){
            while(!q.isEmpty() && nums[i] >= nums[q.peekLast()]){
                q.pollLast();
            }
            q.offerLast(i);
            while(q.peekFirst() <= i - k){
                q.pollFirst();
            }
            res[i - k + 1] = nums[q.peekFirst()];
        }
        return res;
    }
```
# 动态规划
## 思维模式
有两种, 一种是迭代的自底向上的思路, 第二种是递归的自顶向下的思路, 顶代表的是复杂和抽象, 底代表的是简单和具体. 并且可能需要注意重叠子问题用备忘录来解决, 否则可能时间复杂度过高. 

动态规划的核心设计思想是数学归纳法。

相信大家对数学归纳法都不陌生，高中就学过，而且思路很简单。比如我们想证明一个数学结论，那么我们先假设这个结论在 k < n 时成立，然后根据这个假设，想办法推导证明出 k = n 的时候此结论也成立。如果能够证明出来，那么就说明这个结论对于 k 等于任何数都成立。

类似的，我们设计动态规划算法，不是需要一个 dp 数组吗？我们可以假设**dp[0...i-1], 不止是 dp[i-1],记住**都已经被算出来了，然后问自己：怎么通过这些结果算出 dp[i]？

直接拿最长递增子序列这个问题举例你就明白了。不过，首先要定义清楚 dp 数组的含义，即 dp[i] 的值到底代表着什么？

我们的定义是这样的：dp[i] 表示以 nums[i] 这个数结尾的最长递增子序列(https://leetcode.com/problems/longest-increasing-subsequence)的长度。
总结一下如何找到动态规划的状态转移关系：

1、明确 dp 数组的定义。这一步对于任何动态规划问题都很重要，如果不得当或者不够清晰，会阻碍之后的步骤。

2、根据 dp 数组的定义，运用数学归纳法的思想，假设 dp[0...i-1] 都已知，想办法求出 dp[i]，一旦这一步完成，整个题目基本就解决了。

但如果无法完成这一步，很可能就是 dp 数组的定义不够恰当，需要重新定义 dp 数组的含义；或者可能是 dp 数组存储的信息还不够，不足以推出下一步的答案，需要把 dp 数组扩大成二维数组甚至三维数组。

## 思路顺序
举例 https://leetcode.cn/problems/coin-change/
</br>1、确定 base case，这个很简单，显然目标金额 amount 为 0 时算法返回 0，因为不需要任何硬币就已经凑出目标金额了。
</br>2、确定「状态」，也就是原问题和子问题中会变化的变量。由于硬币数量无限，硬币的面额也是题目给定的，只有目标金额会不断地向 base case 靠近，所以唯一的「状态」就是目标金额 amount。有可能状态是多个, 就是多维的, 一般最多是二维. 
</br>3、确定「选择」，也就是导致「状态」产生变化的行为。目标金额为什么变化呢，因为你在选择硬币，你每选择一枚硬币，就相当于减少了目标金额。所以说所有硬币的面值，就是你的「选择」。
</br>4、明确 dp 函数/数组的定义。我们这里讲的是自顶向下的解法，所以会有一个递归的 dp 函数，一般来说函数的参数就是状态转移中会变化的量，也就是上面说到的「状态」；函数的返回值就是题目要求我们计算的量。就本题来说，状态只有一个，即「目标金额」，题目要求我们计算凑出目标金额所需的最少硬币数量。
</br>5. 确定动态规划的公式, 并套入一般情况验证. 
</br>6. 写出初步框架代码, 并估算时间复杂度和空间复杂度, 如果是递归检查是否有重叠子问题, 有的话加备忘录. 
</br>7. 检查初版代码正确后, ac 之后, 检查是否能优化, 一般是降低空间复杂度. 见后续的"如何把空间复杂度降维度?"

## 代码模式
```
# 自顶向下递归的动态规划
def dp(状态1, 状态2, ...):
    for 选择 in 所有可能的选择:
        # 此时的状态已经因为做了选择而改变
        result = 求最值(result, dp(状态1, 状态2, ...))
    return result

# 自底向上迭代的动态规划
# 初始化 base case
dp[0][0][...] = base case
# 进行状态转移
for 状态1 in 状态1的所有取值：
    for 状态2 in 状态2的所有取值：
        for ...
            dp[状态1][状态2][...] = 求最值(选择1，选择2...)
```
## 动态规划相关疑问
* 什么时候该用自顶向下的递归, 或自底向上的迭代
* 最优子结构是什么, 跟动态规划什么关系? 

最优子结构并不是动态规划独有的一种性质，能求最值的问题大部分都具有这个性质；但反过来，最优子结构性质作为动态规划问题的必要条件，一定是让你求最值的，以后碰到那种恶心人的最值题，思路往动态规划想就对了，这就是套路。

动态规划不就是从最简单的 base case 往后推导吗，可以想象成一个链式反应，以小博大。但只有符合最优子结构的问题，才有发生这种链式反应的性质。

找最优子结构的过程，其实就是证明状态转移方程正确性的过程，方程符合最优子结构就可以写暴力解了，写出暴力解就可以看出有没有重叠子问题了，有则优化，无则 OK。这也是套路，经常刷题的读者应该能体会。

* 如何一眼看出重叠子问题

首先，最简单粗暴的方式就是画图，把递归树画出来，看看有没有重复的节点。

* 如何确定动态规划状态变化方程
就是如何确定dp[i] 的含义, 常见的就是直接是题目要求的, 如果行不通, 可能就需要变换思维方式, 绕个弯, 或者提升维度到多维. 
确定好这个公式之后, 如何确定dp[i] 的遍历顺序, 有可能是斜着或者倒着遍历, 一个原则, 就是dp[i] 或 dp[i][j] 依赖的结果要在之前就被遍历出来, 可以画个数组或者二维矩阵帮助理解. 

* 某些测试用例没通过
最常见的就是一些边界没考虑好, 或者初始情况没想清楚或者是笔误了, 因为公式一旦写对就只有这些其他可能了. 

* 如何把空间复杂度降维度
首先把基础的空间复杂度的方案写对, ac 之后, 然后根据代码看当前状态与之前的什么状态有关, 是否只与之前的某个状态有关, 就用一个值代表, 或者之前的某一行状态有关, 就用一行, 从二维空间压缩到一维, 如果用一维不行, 可能就要用一维加一些临时变量作为之前的状态暂存值. 
## 动态规划常见类型题目
* 最长子序列

子序列注意跟子串不一样, 子序列是不连续的, 此类若是一个数组一个字符串, 就是一维, 常规空间复杂度 O(n), 时间复杂度O(n), 空间有可能可以缩减到O(1) 但是要看具体题目;
若是两个数组, 两个字符串, 求公共就是二维, 常规空间复杂度O(mn), 空间有可能可以缩减到O(min(m,n)) 但是要看具体题目, 时间复杂度O(mn), 若不是动态规划, 没有备忘录解决重叠子问题重复计算的话, 时间复杂度就是O(m的n次方), 相当于一颗多叉树, 树高是n, 分叉个数是m, 此类复杂度若要分析一般分析一个最坏时间复杂度即可, 否则太复杂没法分析. 
代码模板如下:
![ccdb8efcdfd459dc0c9710b4fb99e8f](https://user-images.githubusercontent.com/20329409/206965299-8866273f-96f5-4ddc-90e5-7975e3dfeab0.jpg)

* 买卖股票问题 2
https://leetcode.com/problems/best-time-to-buy-and-sell-stock-ii/description/

```
int n = stocks.length;
int [][] res = new int [n][2];
// 0 卖出, 1 持有
res[0][0] = 0;
res[0][1] = -stocks[0];
for(int i = 1; i < n; i++){
    res[i][0] = Math.max(res[i-1][1] + stocks[i], res[i-1][0]);
    res[i][1] = Math.max(res[i-1][1], res[i-1][0] - stocks[i]);
}

return res[n-1][0];

// 下面是节省空间的版本
int n = stocks.length;

// 0 卖出, 1 持有
res[0][0] = 0;
res[0][1] = -stocks[0];

int a = 0;
int b = -stocks[0];
for(int i = 1; i < n; i++){
    int oldA = a
    a = Math.max(b + stocks[i], oldA);
    b = Math.max(b, oldA - stocks[i]);
}

return res[n-1][0];
```
## 背包问题
* 背包定义


那么什么样的问题可以被称作为背包问题？换言之，我们拿到题目如何透过题目的不同包装形式看到里面背包问题的不变内核呢？
我对背包问题定义的理解：</br>
给定一个背包容量target，再给定一个数组nums(物品)，能否按一定方式选取nums中的元素得到target
</br>
1、背包容量target和物品nums的类型可能是数，也可能是字符串

2、target可能题目已经给出(显式)，也可能是需要我们从题目的信息中挖掘出来(非显式)(常见的非显式target比如sum/2等)

3、选取方式有常见的一下几种：每个元素选一次/每个元素选多次/选元素进行排列组合
</br>
那么对应的背包问题就是下面我们要讲的背包分类
</br>

* 背包问题分类
常见的背包类型主要有以下几种：
</br>
1、0/1背包问题：每个元素最多选取一次

2、完全背包问题：每个元素可以重复选择

3、组合背包问题：背包中的物品要考虑顺序

4、分组背包问题：不止一个背包，需要遍历每个背包
</br>

而每个背包问题要求的也是不同的，按照所求问题分类，又可以分为以下几种：
</br>
1、最值问题：要求最大值/最小值

2、存在问题：是否存在…………，满足…………

3、组合问题：求所有满足……的排列组合

* 分类解题模板
背包问题大体的解题模板是两层循环，分别遍历物品nums和背包容量target，然后写转移方程，
根据背包的分类我们确定物品和容量遍历的先后顺序，根据问题的分类我们确定状态转移方程的写法

首先是背包分类的模板：
</br>

1、0/1背包：外循环nums,内循环target,target倒序且target>=nums[i];

2、完全背包：外循环nums,内循环target,target正序且target>=nums[i];

3、组合背包：外循环target,内循环nums,target正序且target>=nums[i];

4、分组背包：这个比较特殊，需要三重循环：外循环背包bags,内部两层循环根据题目的要求转化为1,2,3三种背包类型的模板

然后是问题分类的模板：
</br>

1、最值问题: dp[i] = max/min(dp[i], dp[i-nums]+1)或dp[i] = max/min(dp[i], dp[i-num]+nums);

2、存在问题(bool)：dp[i]=dp[i]||dp[i-num];

3、组合问题：dp[i]+=dp[i-num];

因此把背包类型和问题类型结合起来就会出现以下细分的题目类型：
1、0/1背包最值问题

2、0/1背包存在问题

3、0/1背包组合问题

4、完全背包最值问题

5、完全背包存在问题

6、完全背包组合问题

7、分组背包最值问题

8、分组背包存在问题

9、分组背包组合问题
这九类问题我认为几乎可以涵盖力扣上所有的背包问题

**那如何确定循环的方向呢, 到底是正序遍历还是倒序, 就看要更新的 dp数组的目标 index 的位置, 假设最后结果保存在 dp[i], 但是 dp[i] 由 dp[i-n] 更新, 那么循环的过程中必须保证先更新 dp[i-n] 再更新 dp[i], 否则结果不对**
```
下面看一下具体的题目分析：

1049. 最后一块石头的重量 II
这道题看出是背包问题比较有难度
最后一块石头的重量：从一堆石头中,每次拿两块重量分别为x,y的石头,若x=y,则两块石头均粉碎;若x<y,两块石头变为一块重量为y-x的石头求最后剩下石头的最小重量(若没有剩下返回0)
问题转化为：把一堆石头分成两堆,求两堆石头重量差最小值
进一步分析：要让差值小,两堆石头的重量都要接近sum/2;我们假设两堆分别为A,B,A<sum/2,B>sum/2,若A更接近sum/2,B也相应更接近sum/2
进一步转化：将一堆stone放进最大容量为sum/2的背包,求放进去的石头的最大重量MaxWeight,最终答案即为sum-2*MaxWeight;、
0/1背包最值问题：外循环stones,内循环target=sum/2倒序,应用转移方程1


int lastStoneWeightII(vector<int> &stones)
{
    int sum = accumulate(stones.begin(), stones.end(), 0);
    int target = sum / 2;
    vector<int> dp(target + 1);
    for (int stone : stones)
        for (int i = target; i >= stone; i--)
            dp[i] = max(dp[i], dp[i - stone] + stone);
    return sum - 2 * dp[target];
}
322. 零钱兑换
零钱兑换：给定amount,求用任意数量不同面值的零钱换到amount所用的最少数量
完全背包最值问题：外循环coins,内循环amount正序,应用状态方程1


int coinChange(vector<int> &coins, int amount)
{
    vector<long long> dp(amount + 1, INT_MAX); //给dp数组每个位置赋初值为INT_MAX是为了最后判断是否能填满amount,要用long long 类型
    dp[0] = 0;  //dp[i]:换到面值i所用的最小数量
    for (int coin : coins)
    {
        for (int i = 0; i <= amount; i++)
        {
            if (coin <= i)
                dp[i] = min(dp[i], dp[i - coin] + 1);
        }
    }
    return dp[amount] == INT_MAX ? -1 : dp[amount];
}
416. 分割等和子集
分割等和子集：判断是否能将一个数组分割为两个子集,其和相等
0-1背包存在性问题：是否存在一个子集,其和为target=sum/2,外循环nums,内循环target倒序,应用状态方程2


bool canPartition(vector<int> &nums)
{
    int sum = accumulate(nums.begin(), nums.end(), 0);
    if (sum % 2 == 1)  //如果是和为奇数显然无法分成两个等和子集
        return false;
    int target = sum / 2; 
    vector<int> dp(target + 1, 0); //dp[i]:是否存在子集和为i
    dp[0] = true;   //初始化：target=0不需要选择任何元素，所以是可以实现的
    for (int num : nums)
        for (int i = target; i >= num; i--)
            dp[i] = dp[i] || dp[i - num];
    return dp[target];
}
494. 目标和
目标和：给数组里的每个数字添加正负号得到target
数组和sum,目标和s, 正数和x,负数和y,则x+y=sum,x-y=s,那么x=(s+sum)/2=target
0-1背包不考虑元素顺序的组合问题:选nums里的数得到target的种数,外循环nums,内循环target倒序,应用状态方程3


int findTargetSumWays(vector<int> &nums, int s)
{
    int sum = accumulate(nums.begin(), nums.end(), 0);
    if ((sum + s) % 2 != 0 || sum < s)
        return 0;
    int target = (sum + s) / 2;
    vector<int> dp(target + 1);
    dp[0] = 1;
    for (int num : nums)
        for (int i = target; i >= num; i--)
            dp[i] += dp[i - num];
    return dp[target];
}
279. 完全平方数
完全平方数：对于一个正整数n,找出若干个完全平方数使其和为n,返回完全平方数最少数量
完全背包的最值问题：完全平方数最小为1,最大为sqrt(n),故题目转换为在nums=[1,2.....sqrt(n)]中选任意数平方和为target=n
外循环nums,内循环target正序,应用转移方程1


int numSquares(int n)
{
    vector<int> dp(n + 1, INT_MAX); //dp[i]:和为i的完全平方数的最小数量
    dp[0] = 0;
    for (int num = 1; num <= sqrt(n); num++)
    {
        for (int i = 0; i <= n; i++)
        {
            if (i >= num * num)
                dp[i] = min(dp[i], dp[i - num * num] + 1);
        }
    }
    return dp[n];
}
377. 组合总和 Ⅳ
组合总和IV：在nums中任选一些数,和为target
考虑顺序的组合问题：外循环target,内循环nums,应用状态方程3


int combinationSum4(vector<int> &nums, int target)
{
    vector<int> dp(target + 1);
    dp[0] = 1;
    for (int i = 1; i <= target; i++)
    {
        for (int num : nums)
        {
            if (num <= i) 
                dp[i] += dp[i - num];
        }
    }
    return dp[target];
}
518. 零钱兑换 II
零钱兑换2：任选硬币凑成指定金额,求组合总数
完全背包不考虑顺序的组合问题：外循环coins,内循环target正序,应用转移方程3


int change(int amount, vector<int> &coins)
{
    vector<int> dp(amount + 1);
    dp[0] = 1;
    for (int coin : coins)
        for (int i = 1; i <= amount; i++)
            if (i >= coin)
                dp[i] += dp[i - coin];
    return dp[amount];
}
1155. 掷骰子的N种方法
投掷骰子的方法数：d个骰子,每个有f个面(点数为1,2,...f),求骰子点数和为target的方法
分组0/1背包的组合问题：dp[i][j]表示投掷i个骰子点数和为j的方法数;三层循环：最外层为背包d,然后先遍历target后遍历点数f
应用二维拓展的转移方程3：dp[i][j]+=dp[i-1][j-f];


int numRollsToTarget(int d, int f, int target)
{
    vector<vector<int>> dp(d + 1, vector<int>(target + 1, 0));
    dp[0][0] = 1;
    for (int i = 1; i <= d; i++)
        for (int j = 1; j <= target; j++)
            for (int k = 1; k <= f && j >= k; k++)
                dp[i][j] += dp[i - 1][j - k];
    return dp[d][target];
}

```	
	
	
	
	
	
## 贪心算法
贪心算法可以理解为动态规划的一种特例, 不需要穷举所有情况即可得到最优解. 
什么是贪心选择性质呢，简单说就是：每一步都做出一个局部最优的选择，最终的结果就是全局最优。注意哦，这是一种特殊性质，其实只有一部分问题拥有这个性质。

比如你面前放着 100 张人民币，你只能拿十张，怎么才能拿最多的面额？显然每次选择剩下钞票中面值最大的一张，最后你的选择一定是最优的。

然而，大部分问题明显不具有贪心选择性质。比如打斗地主，对手出对儿三，按照贪心策略，你应该出尽可能小的牌刚好压制住对方，但现实情况我们甚至可能会出王炸

```
// 本文解决一个很经典的贪心算法问题 Interval Scheduling（区间调度问题）, https://leetcode.com/problems/non-overlapping-intervals
// 输入一个区间的集合，请你计算，要想使其中的区间都互不重叠，至少需要移除几个区间？其实等于计算如果区间不重叠, 那么不重叠的区间个数最多是多少, 
// 其实就是按照区间的结束位置升序排列, 选择最早结束的开始作为开始的第一个区间, 那么后续能够容纳的区间长度和个数都是最多的, 证明靠大脑想想和测试用例保证是对的, 略. 
public int eraseOverlapIntervals(int[][] intervals) {
        int [][] t = intervals;
        Arrays.sort(t, (a, b) -> {
            if(a[1] < b[1]){
                return -1;
            }else if(a[1] > b[1]){
                return 1;
            }else {
                return 0;
            }
        });
        int end = t[0][1];
        int cnt = 1 ;
        for(int [] a : t){
            if( a[0] >= end){
                end = a[1];
                cnt++;
            }
        }
        return t.length - cnt;
    }
```
## 题目
```
// https://leetcode.cn/problems/coin-change/
转移方程: 
dp(n, coin[i]) = dp(n-coin[i])+1
   int res = Integer.MAX_VALUE; // 
    public int coinChange(int[] coins, int amount) {
        help(coins,0,amount);
        return res;
    }
    
    public int help(int[] coins,int num, int amount) {
        if(amount == 0) { res = Math.min(res, num);}
        if(amount < 0) return -1;
        for(int c: coins){
            int method = help(coins,num+1, amount-c);
            if(method < 0) continue;
        }
}

```
```
// https://leetcode.cn/problems/coin-change-2/
int [][] dp = new int [coins.length + 1][amount + 1];
int change(int amount, int[] coins){
    for(int i = 1; i< coins.length; i++){
        dp[i][0] = 1;
    }
    for(int i = 1; i < coins.length + 1; i++){
        for(int j = 1; j <= amount; j++){
            if(coins[i-1] > j){
                dp[i][j] = dp[i-1][j];
            }else{
                dp[i][j] = dp[i][j-coins[i - 1]] + dp[i-1][j];
            }
        }
    }
    return dp[coins.length][amount];
}

// https://leetcode.cn/problems/coin-change-2/, save space
int [] dp = new int [amount + 1];
int change(int amount, int[] coins){
    dp[0] = 1;
    for(int i = 1; i < coins.length + 1; i++){
        for(int j = 1; j <= amount; j++){
            if(coins[i-1] <= j){
                dp[j] = dp[j-coins[i - 1]] + dp[j];
            }
        }
    }
    return dp[coins.length][amount];
}
```
```
// 最长回文子序列, save space
  for (int i = n - 2; i >= 0; i--) {
        int tmp = 0;
        for (int j = i + 1; j < n; j++) {
           int tmp = dp[j];
            // 状态转移方程
            if (s[i] == s[j])
                dp[j] = pre + 2;
            else
                dp[j] = max(dp[j], dp[j - 1]);
            pre = tmp;
        }
    }
```
## 二维动态规划
#### https://leetcode.com/problems/unique-paths/submissions/
1. 最优解法, 解法1.
一维数组, 实际上就是选择 x,y 中最小的那一个作为一维数组的长度, 然后逐行遍历上去, arr[i] += arr[i-1]
2. 组合方法, x,y 可以理解成总步数 x+y-2, 中选择x-1 步往右走, 总共多少种组合方式, 例如: 右右右下下下, 可见是不分顺序的, x+y-2的阶乘除以 (x-1)的阶乘再除以 (y-1)的阶乘.
3. 二维数组方法, 最常见
4. 类似图论里的深度搜索, 自底向上, 因为向左或者向右走可以抽象成树的左节点或右节点, 根节点为出发点, 那么树的深度就是时间复杂度是 2的(x+y-2)的次方, 时间复杂度过高, 空间复杂度是栈的深度: x+y-2.
5. 传统的递归, fn(x,y) = fn(x-1,y) + fn(x,y-1) 跟4的时间和空间复杂度是一样的, 只是是自顶向下的, 上层代表着抽象和复杂, 下层代表着具体和简单. 
6. 递归里面用记忆矩阵, 因为递归中有节点是被重复计算了的. 


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
# 区间相关问题

所谓区间问题，就是线段问题，让你合并所有线段、找出线段的交集等等。主要有两个技巧：

1、排序。常见的排序方法就是按照区间起点排序，或者先按照起点升序排序，若起点相同，则按照终点降序排序。当然，如果你非要按照终点排序，无非对称操作，本质都是一样的。

2、画图。就是说不要偷懒，勤动手，两个区间的相对位置到底有几种可能，不同的相对位置我们的代码应该怎么去处理。

3. 但是区间调度问题，是属于动态规划里贪心的算法范畴, 题目里要求最多, 最大, 最值的字眼, 例如 leetcode 435 需要按end排序，以便满足贪心选择性质。对于区间合并问题，其实按end和start排序都可以，不过为了清晰起见，我们选择按start排序。

4. 两个区间之间, 只有三种情况, 一种是包含, 一种是相交, 另一种是不相交, 没有其他了, 所以只要能写出任何一种情况, 取反就能得到另外两种情况, 反之亦然. 

## 区间相关面试问题
求一个在线系统的某个时间段的最大在线人数和某个时刻的在线人数，给出的条件是一个日志数组 loga ，每个数组元素有三个属性，一个是用户id，一个是登陆时间 s，一个是登出时间 e。
不需要排序，用空间换时间，用一个时间数组 ta ，假设一天 x 秒，那么就是一个x 个元素的数组，遍历日志数组，
for l : loga{
    ta[s]++;
    ta[e]--;
}
某个时间点tx 的在线人数就是sum(ta[t], t从0 到tx),
某个时间段 t1 - t2的最大在线人数，
for t=t1;t <= t2; t++{
    用第一个式子求出ta[t1] 之后，再依次累加ta[t] , 并保留最大值即可.
}
```
// leetcode 1288 删除被覆盖的区间, 返回删除后, 剩余区间的数目
// 按起点升序排序, 再按终点降序排序. 
int removeCoveredIntervals(int[][] intvs) {
    // 按照起点升序排列，起点相同时降序排列
    Arrays.sort(intvs, (a, b) -> {
        if (a[0] == b[0]) {
            return b[1] - a[1];
        }
        return a[0] - b[0]; 
    });

    // 记录合并区间的起点和终点
    int left = intvs[0][0];
    int right = intvs[0][1];

    int res = 0;
    for (int i = 1; i < intvs.length; i++) {
        int[] intv = intvs[i];
        // 情况一，找到覆盖区间
        if (left <= intv[0] && right >= intv[1]) {
            res++;
        }
        // 情况二，找到相交区间，合并
        if (right >= intv[0] && right <= intv[1]) {
            right = intv[1];
        }
        // 情况三，完全不相交，更新起点和终点
        if (right < intv[0]) {
            left = intv[0];
            right = intv[1];
        }
    }

    return intvs.length - res;
}
```

```
// https://leetcode.cn/problems/interval-list-intersections/description/, 两个区间列表的交集

   public int[][] intervalIntersection(int[][] firstList, int[][] secondList) {
        LinkedList<int []> list = new LinkedList<>();
        int [][] list1 = firstList;
        int [][] list2 = secondList;
        int i = 0;
        int j = 0;
        while( i < firstList.length && j < secondList.length ){
            int [] ii = list1[i];
            int [] jj = list2[j];
            int [] cur = new int [2];
            if(!((ii[1] < jj[0]) || (ii[0] > jj[1] ))){ // 包含和相交都属于有交集, 是不相交的取反
                cur[0] = Math.max(ii[0], jj[0]);
                cur[1] = Math.min(ii[1], jj[1]);
                list.add(cur);
            }
            if(ii[1] < jj[1]){ // 只要ii 区间的end 小于 jj 说明 ii 就可以前进了, 画图就好了, 就几种情况而已
                i++;
            }else{
                j++;
            }
        }
        int [][] res = new int [list.size()][2];
        int k = 0;
        for(int [] tmp: list){
            res[k] = list.get(k);
            k++;
        }
        return res;
    }

```
#  设计一个常数时间插入、删除和获取随机元素的数据结构

```

// 力扣第 380 题
class RandomizedSet {
    HashMap<Integer, Integer> map;
    ArrayList<Integer> list;
    Random random;
    public RandomizedSet() {
        random = new Random();
        list = new ArrayList<>();
        map = new HashMap<>();
    }
    
    public boolean insert(int val) {
        if(map.containsKey(val)){
            return false;
        }
        list.add(val);
        int index = list.size() - 1;
        map.put(val, index);
        return true;
    }
    
    public boolean remove(int val) {
        if(!map.containsKey(val)){
            return false;
        }
        int index = map.remove(val);
        if(index < list.size() - 1){
            int lastEle = list.get(list.size() - 1);
            list.set(index, lastEle);
            map.put(lastEle, index);
        }
        list.remove(list.size() - 1);
        return true;
    }
    
    public int getRandom() {
        return list.get(random.nextInt(list.size()));
    }
}

```
类似题目
```
// https://leetcode.com/problems/random-pick-with-blacklist/description/

   Random r = new Random();
   int board = 0;
     HashMap<Integer, Integer> map;
    public Solution(int n, int[] blacklist) {
        board = n - blacklist.length;
       map = new HashMap<>();
        for(int b : blacklist){
            map.put(b, -1);
        }
        int last = n - 1;
        for(int b: blacklist){
            if(b >= board){
                continue;
            }
            while(map.containsKey(last)){
                last--;
            }
            map.put(b, last--);
        }
    }
    
    public int pick() {
        int ran = r.nextInt(board);
        Integer val = map.get(ran);
        if(val != null){
            return val;
        }
        return ran;
    }
```



# 回溯

形式一、元素无重不可复选，即 nums 中的元素都是唯一的，每个元素最多只能被使用一次，这也是最基本的形式。

以组合为例，如果输入 nums = [2,3,6,7]，和为 7 的组合应该只有 [7]。

形式二、元素可重不可复选，即 nums 中的元素可以存在重复，每个元素最多只能被使用一次。

以组合为例，如果输入 nums = [2,5,2,1,2]，和为 7 的组合应该有两种 [2,2,2,1] 和 [5,2]。

形式三、元素无重可复选，即 nums 中的元素都是唯一的，每个元素可以被使用若干次。

以组合为例，如果输入 nums = [2,3,6,7]，和为 7 的组合应该有两种 [2,2,3] 和 [7]。

```
// 回溯算法框架

List<List<Integer>> result = new LinkedList<>();
LinkedList<Integer> list = new LinkedList<>();


void backtrack(int [] nums):
    if 满足结束条件:
        result.add(new ArrayList<>(list));
        return
    
    for n in nums:
        做选择
        backtrack(nums)
        撤销选择
```

* 实战题目

```
// https://leetcode.cn/problems/combination-sum-ii/

给定一个候选人编号的集合 candidates 和一个目标数 target ，找出 candidates 中所有可以使数字和为 target 的组合。

candidates 中的每个数字在每个组合中只能使用 一次 。

注意：解集不能包含重复的组合。 

 

示例 1:

输入: candidates = [10,1,2,7,6,1,5], target = 8,
输出:
[
[1,1,6],
[1,2,5],
[1,7],
[2,6]
]


class Solution {
    List<List<Integer>> res = new LinkedList<>();
    LinkedList<Integer> list = new LinkedList<>();
    public List<List<Integer>> combinationSum2(int[] candidates, int target) {
        Arrays.sort(candidates);
        help(candidates, target, 0);
        return res;
    }

    void help(int[] candidates, int t, int start){
        if( t == 0){
            res.add(new ArrayList<>(list));
            return;
        }
        for(int i = start; i < candidates.length; i++){
            if(i != start && candidates[i] == candidates[i-1]){
                continue;
            }
            if(candidates[i] > t){
                return;
            }
            t -= candidates[i];
            list.add(candidates[i]);
            help(candidates, t, i+1);
            t += candidates[i];
            list.removeLast();
        }
    }
}


// https://leetcode.cn/problems/permutations-ii/description/

给定一个可包含重复数字的序列 nums ，按任意顺序 返回所有不重复的全排列。

 

示例 1：

输入：nums = [1,1,2]
输出：
[[1,1,2],
 [1,2,1],
 [2,1,1]]


class Solution {
   List<List<Integer>> res = new LinkedList<>();
LinkedList<Integer> track = new LinkedList<>();
boolean[] used;

public List<List<Integer>> permuteUnique(int[] nums) {
    // 先排序，让相同的元素靠在一起
    Arrays.sort(nums);
    used = new boolean[nums.length];
    backtrack(nums);
    return res;
}

void backtrack(int[] nums) {
    if (track.size() == nums.length) {
        res.add(new LinkedList(track));
        return;
    }

    for (int i = 0; i < nums.length; i++) {
        if (used[i]) {
            continue;
        }
        // 新添加的剪枝逻辑，固定相同的元素在排列中的相对位置
        // 当出现重复元素时，比如输入 nums = [1,2,2',2'']，2' 只有在 2 已经被使用的情况下才会被选择，
        // 同理，2'' 只有在 2' 已经被使用的情况下才会被选择，这就保证了相同元素在排列中的相对位置保证固定。
        if (i > 0 && nums[i] == nums[i - 1] && !used[i - 1]) {
            continue;
        }
        track.add(nums[i]);
        used[i] = true;
        backtrack(nums);
        track.removeLast();
        used[i] = false;
    }
}
}

// https://leetcode.cn/problems/permutations/
给定一个不含重复数字的数组 nums ，返回其 所有可能的全排列 。你可以 按任意顺序 返回答案。

 

示例 1：

输入：nums = [1,2,3]
输出：[[1,2,3],[1,3,2],[2,1,3],[2,3,1],[3,1,2],[3,2,1]]

class Solution {
        List<List<Integer>> res = new LinkedList<>();

    LinkedList<Integer> list = new LinkedList<>();
    boolean [] used ;
    public List<List<Integer>> permute(int[] nums) {
        used = new boolean[nums.length];
        help(nums);
        return res;
    }

    void help(int [] nums){
        if(list.size() == nums.length){
            res.add(new ArrayList<>(list));
            return;
        }

        for(int i = 0; i < nums.length; i++){
            if(used[i]){
                continue;
            }
            list.add(nums[i]);
            used[i] = true;
            help(nums);
            list.removeLast();
            used[i] = false;
        }
    }
}


https://leetcode.cn/problems/combination-sum/description/

给你一个 无重复元素 的整数数组 candidates 和一个目标整数 target ，找出 candidates 中可以使数字和为目标数 target 的 所有 不同组合 ，并以列表形式返回。你可以按 任意顺序 返回这些组合。

candidates 中的 同一个 数字可以 无限制重复被选取 。如果至少一个数字的被选数量不同，则两种组合是不同的。 

对于给定的输入，保证和为 target 的不同组合数少于 150 个。

 

示例 1：

输入：candidates = [2,3,6,7], target = 7
输出：[[2,2,3],[7]]
解释：
2 和 3 可以形成一组候选，2 + 2 + 3 = 7 。注意 2 可以使用多次。
7 也是一个候选， 7 = 7 。
仅有这两种组合。

class Solution {
        List<List<Integer>> res = new LinkedList<>();
    LinkedList<Integer> list = new LinkedList<>();
    public List<List<Integer>> combinationSum(int[] candidates, int target) {
        Arrays.sort(candidates);
        help(candidates, target, 0);
        return res;
    }

    void help(int[] candidates, int t, int start){
        if( t == 0){
            res.add(new ArrayList<>(list));
            return;
        }
        for(int i = start; i < candidates.length; i++){
            if(candidates[i] > t){
                return;
            }
            t -= candidates[i];
            list.add(candidates[i]);
            help(candidates, t, i); // 重点: 从i 开始即可, 不是 i+1 . 
            t += candidates[i];
            list.removeLast();
        }
    }

 
}
```

来回顾一下排列/组合/子集问题的三种形式在代码上的区别。

由于子集问题和组合问题本质上是一样的，无非就是 base case 有一些区别，所以把这两个问题放在一起看。

```
形式一、元素无重不可复选，即 nums 中的元素都是唯一的，每个元素最多只能被使用一次，backtrack 核心代码如下：

/* 组合/子集问题回溯算法框架 */
void backtrack(int[] nums, int start) {
    // 回溯算法标准框架
    for (int i = start; i < nums.length; i++) {
        // 做选择
        track.addLast(nums[i]);
        // 注意参数
        backtrack(nums, i + 1);
        // 撤销选择
        track.removeLast();
    }
}

/* 排列问题回溯算法框架 */
void backtrack(int[] nums) {
    for (int i = 0; i < nums.length; i++) {
        // 剪枝逻辑
        if (used[i]) {
            continue;
        }
        // 做选择
        used[i] = true;
        track.addLast(nums[i]);

        backtrack(nums);
        // 撤销选择
        track.removeLast();
        used[i] = false;
    }
}
形式二、元素可重不可复选，即 nums 中的元素可以存在重复，每个元素最多只能被使用一次，其关键在于排序和剪枝，backtrack 核心代码如下：

Arrays.sort(nums);
/* 组合/子集问题回溯算法框架 */
void backtrack(int[] nums, int start) {
    // 回溯算法标准框架
    for (int i = start; i < nums.length; i++) {
        // 剪枝逻辑，跳过值相同的相邻树枝
        if (i > start && nums[i] == nums[i - 1]) {
            continue;
        }
        // 做选择
        track.addLast(nums[i]);
        // 注意参数
        backtrack(nums, i + 1);
        // 撤销选择
        track.removeLast();
    }
}


Arrays.sort(nums);
/* 排列问题回溯算法框架 */
void backtrack(int[] nums) {
    for (int i = 0; i < nums.length; i++) {
        // 剪枝逻辑
        if (used[i]) {
            continue;
        }
        // 剪枝逻辑，固定相同的元素在排列中的相对位置
        if (i > 0 && nums[i] == nums[i - 1] && !used[i - 1]) {
            continue;
        }
        // 做选择
        used[i] = true;
        track.addLast(nums[i]);

        backtrack(nums);
        // 撤销选择
        track.removeLast();
        used[i] = false;
    }
}
形式三、元素无重可复选，即 nums 中的元素都是唯一的，每个元素可以被使用若干次，只要删掉去重逻辑即可，backtrack 核心代码如下：

/* 组合/子集问题回溯算法框架 */
void backtrack(int[] nums, int start) {
    // 回溯算法标准框架
    for (int i = start; i < nums.length; i++) {
        // 做选择
        track.addLast(nums[i]);
        // 注意参数
        backtrack(nums, i);
        // 撤销选择
        track.removeLast();
    }
}


/* 排列问题回溯算法框架 */
void backtrack(int[] nums) {
    for (int i = 0; i < nums.length; i++) {
        // 做选择
        track.addLast(nums[i]);
        backtrack(nums);
        // 撤销选择
        track.removeLast();
    }
}
```

# 二叉树
* 编程模式
二叉树题目的递归解法可以分两类思路，第一类是遍历一遍二叉树得出答案，第二类是通过分解问题计算出答案，这两类思路分别对应着 回溯算法核心框架 和 动态规划核心框架。 
例如下面这个就是类似回溯算法的模式.然后想代码应该写在前序位置还是中序, 还是后序, 剩下的交给递归模式去帮你完成, 用递归的思想去解决. 
综上，遇到一道二叉树的题目时的通用思考过程是：

1、是否可以通过遍历一遍二叉树得到答案？如果可以，用一个 traverse 函数配合外部变量来实现。

2、是否可以定义一个递归函数，通过子问题（子树）的答案推导出原问题的答案？如果可以，写出这个递归函数的定义，并充分利用这个函数的返回值。

3、无论使用哪一种思维模式，你都要明白二叉树的每一个节点需要做什么，需要在什么时候（前中后序）做。

那前序位置和后序位置如何区别呢, 前序位置是刚刚进入节点的时刻，后序位置是即将离开节点的时刻。
但这里面大有玄妙，意味着前序位置的代码只能从函数参数中获取父节点传递来的数据，而后序位置的代码不仅可以获取参数数据，还可以获取到子树通过函数返回值传递回来的数据。那么换句话说，**一旦你发现题目和子树有关，那大概率要给函数设置合理的定义和返回值，在后序位置写代码了。**


```
void traverse(TreeNode root) {
    // 前序位置
    traverse(root.left);
    // 中序位置
    traverse(root.right);
    // 后序位置
}
```

二叉树深度可以写出以上所说的一种是回溯递归的模式, 第二种是分解, 把复杂问题简单化分解的方式
第一种: 
```
// 记录最大深度
int res = 0;
int depth = 0;

// 主函数
int maxDepth(TreeNode root) {
	traverse(root);
	return res;
}

// 二叉树遍历框架
void traverse(TreeNode root) {
	if (root == null) {
		// 到达叶子节点
		res = Math.max(res, depth);
		return;
	}
	// 前序遍历位置
	depth++;
	traverse(root.left);
	traverse(root.right);
	// 后序遍历位置
	depth--;
}
```

第二种:
```
int maxDepth(TreeNode root) {
	if (root == null) {
		return 0;
	}
	// 递归计算左右子树的最大深度
	int leftMax = maxDepth(root.left);
	int rightMax = maxDepth(root.right);
	// 整棵树的最大深度
	int res = Math.max(leftMax, rightMax) + 1;

	return res;
}
```

## 二叉树的遍历
最简单的划分：是深度优先（先访问子节点，再访问父节点，最后是第二个子节点）？还是广度优先（先访问第一个子节点，再访问第二个子节点，最后访问父节点）？ 深度优先可进一步按照根节点相对于左右子节点的访问先后来划分。如果把左节点和右节点的位置固定不动，那么根节点放在左节点的左边，称为前序（pre-order）、根节点放在左节点和右节点的中间，称为中序（in-order）、根节点放在右节点的右边，称为后序（post-order）。对广度优先而言，遍历没有前序中序后序之分：给定一组已排序的子节点，其“广度优先”的遍历只有一种唯一的结果。

## 二叉树的分类
满二叉树, 完全二叉树, 平衡二叉树
![enter image description here](https://drive.google.com/uc?id=1WvdQnszfoBQkf94qiUDM2W_a4BWwArD8)
深度为k, 拥有2的k+1次方减一的节点的为满二叉树, 每一层的节点数都是最大节点数; 而若最后一层不是满的, 其余层都是满的, 则为完全二叉树, 具有n个节点的完全二叉树的深度为log2n +1(以2为底数)。深度为k的完全二叉树, 最少有2的k次方个节点, 最多有2的k+1 次方减一个节点. 
## 二叉树的构造
二叉树的构造问题一般都是使用「分解问题」的思路：构造整棵树 = 根节点 + 构造左子树 + 构造右子树。先找出根节点，然后根据根节点的值找到左右子树的元素，进而递归构建出左右子树。

前中序, 后中序能完全确定一颗二叉树, 但是前后序不能, 因为我们假设前序遍历的第二个元素是左子树的根节点，但实际上左子树有可能是空指针，那么这个元素就应该是右子树的根节点。由于这里无法确切进行判断，所以导致了最终答案的不唯一。
前中序和后中序构造, 都是要求出左子树的节点个数, 并且终止条件是 前序(后序) 的指针, if preStart > preEnd , return null; 画一个非完全二叉树, 构造出前中序序列, 即可, 左右子树的preStart, 和preEnd 都是用根节点的pre 索引, 不会用到 inorder 的索引.

```
// 中后序构造
// 存储 inorder 中值到索引的映射
HashMap<Integer, Integer> valToIndex = new HashMap<>();

TreeNode buildTree(int[] inorder, int[] postorder) {
    for (int i = 0; i < inorder.length; i++) {
        valToIndex.put(inorder[i], i);
    }
    return build(inorder, 0, inorder.length - 1,
                 postorder, 0, postorder.length - 1);
}

/* 
    build 函数的定义：
    后序遍历数组为 postorder[postStart..postEnd]，
    中序遍历数组为 inorder[inStart..inEnd]，
    构造二叉树，返回该二叉树的根节点 
*/
TreeNode build(int[] inorder, int inStart, int inEnd,
               int[] postorder, int postStart, int postEnd) {
    // root 节点对应的值就是后序遍历数组的最后一个元素
    int rootVal = postorder[postEnd];
    // rootVal 在中序遍历数组中的索引
    int index = valToIndex.get(rootVal);

    TreeNode root = new TreeNode(rootVal);
    // 递归构造左右子树
    root.left = build(preorder, ?, ?,
                      inorder, ?, ?);

    root.right = build(preorder, ?, ?,
                       inorder, ?, ?);
    return root;
}
```

## 二叉树的题目
```
// 二叉树的最大直径
int res = Integer.MAX_VALUE;

int longestPath(TreeNode node){
   maxDeep(node);
   return res;
}

int maxDeep(TreeNode node){
     if(node == null) return 0 ;
    int left = longestPath(node.left);
    int right = longestPath(node.right);
    res = Math.max(res,left+right+1);
    return left > right ? left: right;
}
```
```
// https://leetcode.com/problems/find-largest-value-in-each-tree-row/, 递归两种解法中的遍历思路
        public List<Integer> largestValues(TreeNode root) {
        List<Integer> res = new ArrayList<Integer>();
        helper(root, res, 0);
        return res;
    }

    
    private void helper(TreeNode root, List<Integer> res, int d){
        if(root == null){
            return;
        }
        if(d == res.size()){
            res.add(root.val);
        }else{
        res.set(d, Math.max(res.get(d), root.val));
        }
        helper(root.left, res, d+1);
        helper(root.right, res, d+1);
    }
```
## 二叉树最近公共祖先节点
为什么要写这样一个奇怪的find函数呢?
如果在前序位置做是否找到的判断就可以提前返回不需要找后续了, 如果在后序位置就要遍历所有节点. 
```
// 二叉树最近公共祖先节点的代码基础模板, 最近公共祖先系列问题的解法都是把这个函数作为框架的
// 定义：在以 root 为根的二叉树中寻找值为 val1 或 val2 的节点
TreeNode find(TreeNode root, int val1, int val2) {
    // base case
    if (root == null) {
        return null;
    }
    // 前序位置，看看 root 是不是目标值
    if (root.val == val1 || root.val == val2) {
        return root;
    }
    // 去左右子树寻找
    TreeNode left = find(root.left, val1, val2);
    TreeNode right = find(root.right, val1, val2);
    // 后序位置，已经知道左右子树是否存在目标值
    return left != null ? left : right;
}
```


```
先来看看力扣第 236 题「二叉树的最近公共祖先」：

给你输入一棵不含重复值的二叉树，以及存在于树中的两个节点p和q，请你计算p和q的最近公共祖先节点。

TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
    return find(root, p.val, q.val);
}

// 在二叉树中寻找 val1 和 val2 的最近公共祖先节点
TreeNode find(TreeNode root, int val1, int val2) {
    if (root == null) {
        return null;
    }
    // 前序位置
    if (root.val == val1 || root.val == val2) {
        // 如果遇到目标值，直接返回
        return root;
    }
    TreeNode left = find(root.left, val1, val2);
    TreeNode right = find(root.right, val1, val2);
    // 后序位置，已经知道左右子树是否存在目标值
    if (left != null && right != null) {
        // 当前节点是 LCA 节点
        return root;
    }

    return left != null ? left : right;
}
```


```
比如力扣第 1676 题「二叉树的最近公共祖先 IV」：

依然给你输入一棵不含重复值的二叉树，但这次不是给你输入p和q两个节点了，而是给你输入一个包含若干节点的列表nodes（这些节点都存在于二叉树中），让你算这些节点的最近公共祖先。

TreeNode lowestCommonAncestor(TreeNode root, TreeNode[] nodes) {
    // 将列表转化成哈希集合，便于判断元素是否存在
    HashSet<Integer> values = new HashSet<>();
    for (TreeNode node : nodes) {
        values.add(node.val);
    }

    return find(root, values);
}

// 在二叉树中寻找 values 的最近公共祖先节点
TreeNode find(TreeNode root, HashSet<Integer> values) {
    if (root == null) {
        return null;
    }
    // 前序位置
    if (values.contains(root.val)){
        return root;
    }

    TreeNode left = find(root.left, values);
    TreeNode right = find(root.right, values);
    // 后序位置，已经知道左右子树是否存在目标值
    if (left != null && right != null) {
        // 当前节点是 LCA 节点
        return root;
    }

    return left != null ? left : right;
}

```

```
比如力扣第 1644 题「二叉树的最近公共祖先 II」：

给你输入一棵不含重复值的二叉树的，以及两个节点p和q，如果p或q不存在于树中，则返回空指针，否则的话返回p和q的最近公共祖先节点。

// 用于记录 p 和 q 是否存在于二叉树中
boolean foundP = false, foundQ = false;

TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
    TreeNode res = find(root, p.val, q.val);
    if (!foundP || !foundQ) {
        return null;
    }
    // p 和 q 都存在二叉树中，才有公共祖先
    return res;
}

// 在二叉树中寻找 val1 和 val2 的最近公共祖先节点
TreeNode find(TreeNode root, int val1, int val2) {
    if (root == null) {
        return null;
    }
    TreeNode left = find(root.left, val1, val2);
    TreeNode right = find(root.right, val1, val2);

    // 后序位置，判断当前节点是不是 LCA 节点
    if (left != null && right != null) {
        return root;
    }

    // 后序位置，判断当前节点是不是目标值
    if (root.val == val1 || root.val == val2) {
        // 找到了，记录一下
        if (root.val == val1) foundP = true;
        if (root.val == val2) foundQ = true;
        return root;
    }

    return left != null ? left : right;
}
```

```
看力扣第 235 题「二叉搜索树的最近公共祖先」：

给你输入一棵不含重复值的二叉搜索树，以及存在于树中的两个节点p和q，请你计算p和q的最近公共祖先节点。

TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
    // 保证 val1 较小，val2 较大
    int val1 = Math.min(p.val, q.val);
    int val2 = Math.max(p.val, q.val);
    return find(root, val1, val2);
}

// 在 BST 中寻找 val1 和 val2 的最近公共祖先节点
TreeNode find(TreeNode root, int val1, int val2) {
    if (root == null) {
        return null;
    }
    if (root.val > val2) {
        // 当前节点太大，去左子树找
        return find(root.left, val1, val2);
    }
    if (root.val < val1) {
        // 当前节点太小，去右子树找
        return find(root.right, val1, val2);
    }
    // val1 <= root.val <= val2
    // 则当前节点就是最近公共祖先
    return root;
}
```
## 二杈搜索树(binary search tree, BST)
左子树的所有节点都小于根节点, 右子树的所有节点都大于根节点的二叉树, 同样左子树和右子树也都是BST.

中序遍历可以得到递增序列. 反之可以得到递减序列
```
// 递增
void traverse(TreeNode root){
    traverse(root.left);
    print(root.val);
    traverse(root.right);
}

// 递减
void traverse(TreeNode root){
    traverse(root.right);
    print(root.val);
    traverse(root.left);
}
```

```
// 删除某个二叉搜索树的节点
// 如果左右子节点都非空, 则找到左子树最大或者右子树最小(右子树里面最左边的节点) 即为替代节点.
TreeNode deleteNode(TreeNode root, int key) {
    if (root == null) return null;
    if (root.val == key) {
        // 这两个 if 把情况 1 和 2 都正确处理了
        if (root.left == null) return root.right;
        if (root.right == null) return root.left;
        // 处理情况 3
        // 获得右子树最小的节点
        TreeNode minNode = getMin(root.right);
        // 删除右子树最小的节点
        root.right = deleteNode(root.right, minNode.val);
        // 用右子树最小的节点替换 root 节点
        minNode.left = root.left;
        minNode.right = root.right;
        root = minNode;
    } else if (root.val > key) {
        root.left = deleteNode(root.left, key);
    } else if (root.val < key) {
        root.right = deleteNode(root.right, key);
    }
    return root;
}

TreeNode getMin(TreeNode node) {
    // BST 最左边的就是最小的
    while (node.left != null) node = node.left;
    return node;
}
```
```
// 不同二叉树的个数, leetcode 96
class Solution {
    int [][] memo ; 
    public int numTrees(int n) {
        memo = new int [n+1][n+1];
       return count(1, n);
    }

    int count(int low, int high){
        if(low >= high) return 1;
        if(memo[low][high] != 0) return memo[low][high];
        int res = 0 ;
        for(int i =low; i<=high;i++){
            int l = count(low, i-1);
            int r = count(i+1, high);
            res += l*r;
        }
        memo[low][high] = res;
        return res;
    }
}
```
## 完全二叉树

完全二叉树如下图，每一层都是紧凑靠左排列的, 中间没有空的节点
```
// 计算完全二叉树的节点个数, 时间复杂度是O(logN * logN), N 为树的总节点个数, 因为递归的深度是 logN, while 循环的次数也是树高.
public int countNodes(TreeNode root) {
    TreeNode l = root, r = root;
    // 沿最左侧和最右侧分别计算高度
    int hl = 0, hr = 0;
    while (l != null) {
        l = l.left;
        hl++;
    }
    while (r != null) {
        r = r.right;
        hr++;
    }
    // 如果左右侧计算的高度相同，则是一棵满二叉树
    if (hl == hr) {
        return (int)Math.pow(2, hl) - 1;
    }
    // 如果左右侧的高度不同，则按照普通二叉树的逻辑计算
    return 1 + countNodes(root.left) + countNodes(root.right);
}
```
# AVL 树
平衡二叉搜索树（Self-balancing binary search tree）又被称为AVL树（有别于AVL算法），且具有以下性质：它是一棵空树或它的左右两个子树的高度差的绝对值不超过1，并且左右两个子树都是一棵平衡二叉树。当只有一个根节点时, 根节点的高度定义为1.

平衡二叉树就是二叉树的构建过程中，每当插入一个结点，看是不是因为树的插入破坏了树的平衡性，若是，则找出最小不平衡树。在保持二叉树特性的前提下，调整最小不平衡子树中各个结点之间的链接关系，进行相应的旋转，使之成为新的平衡子树


**AVL**  **效率总结 :** 查找的时间复杂度维持在O(logN)，不会出现最差情况
AVL树在执行每个插入操作时最多需要1次旋转，其时间复杂度在O(logN)左右。
AVL树在执行删除时代价稍大，执行每个删除操作的时间复杂度需要O(2logN)。
* 红黑树
红黑树相对于AVL树来说，牺牲了部分平衡性以换取插入/删除操作时少量的旋转操作，整体来说性能要优于AVL树。

* AVL VS 红黑树
avl 更加平衡, 查找效率更高,但是插入和删除所需要的旋转代价也更高, 红黑树平均查找时间复杂度O(lgn), 最坏是


# B树
https://en.wikipedia.org/wiki/B-tree

According to Knuth's definition, a B-tree of order _m_ is a tree which satisfies the following properties:
1.  Every node has at most  _m_  children.
2.  Every non-leaf node (except root) has at least ⌈_m_/2⌉ child nodes.
3.  The root has at least two children if it is not a leaf node.
4.  A non-leaf node with  _k_  children contains  _k_  − 1 keys.
5.  All leaves appear in the same level.

![enter image description here](https://drive.google.com/uc?id=1EeNGv03evzr7zCnjHKEKAl_rgV0ebVCH)

B树相对于平衡二叉树的不同是，每个节点包含的关键字增多了，特别是在B树应用到数据库中的时候，数据库充分利用了磁盘块的原理（磁盘数据存储是采用块的形式存储的，每个块的大小为4K，每次IO进行数据读取时，同一个磁盘块的数据可以一次性读取出来）把节点大小限制和充分使用在磁盘快大小范围；把树的节点关键字增多后树的深度比原来的二叉树少了，减少数据查找的次数和复杂度;

# B+ 树
![enter image description here](https://drive.google.com/uc?id=1F_s695E7TcNkqRhdPszORZQt7sCdplTU)
1.  节点的子树数和关键字数相同（B 树是关键字数比子树数少一）
2.  节点的关键字表示的是子树中的最大数，在子树中同样含有这个数据(根节点的最大关键字其实就表示整个 B+ 树的最大元素。)
3.  叶子节点包含了全部数据，同时符合左小右大的顺序
4. 叶子节点存有指向下个叶子节点的指针, 适合范围查询. 

* 为什么B+ 更适合做数据库和文件系统的数据类型呢
https://stackoverflow.com/questions/870218/differences-between-b-trees-and-b-trees
B+ 树的非叶子节点不需要指针去指向数据, 也就是说比B 树有更多的指针能指向子节点, 可以理解为整个树可能高度更低, 需要的磁盘io 更少,  而且叶子节点有指向下个节点的指针, 不需要再从顶向下扫描周边叶子节点, 适合范围查询.


# 排序
* 稳定排序 vs 不稳定排序

还有一点需要注意的是，快速排序是「不稳定排序」，与之相对的，前文讲的 归并排序 是「稳定排序」。

对于序列中的相同元素，如果排序之后它们的相对位置没有发生改变，则称该排序算法为「稳定排序」，反之则为「不稳定排序」。

如果单单排序 int 数组，那么稳定性没有什么意义。但如果排序一些结构比较复杂的数据，那么稳定性排序就有更大的优势了。

比如说你有若干订单数据，已经按照订单号排好序了，现在你想对订单的交易日期再进行排序：

如果用稳定排序算法（比如归并排序），那么这些订单不仅按照交易日期排好了序，而且相同交易日期的订单的订单号依然是有序的。

但如果你用不稳定排序算法（比如快速排序），那么虽然排序结果会按照交易日期排好序，但相同交易日期的订单的订单号会丧失有序性。

在实际工程中我们经常会将一个复杂对象的某一个字段作为排序的 key，所以应该关注编程语言提供的 API 底层使用的到底是什么排序算法，是稳定的还是不稳定的，这很可能影响到代码执行的效率甚至正确性。

* 快速排序为什么是不稳定的

快速排序是不稳定的。这个论述在标准的教科书上已经提了很多次，也没有什么疑问。
但是，可否通过改进的方法将其变成一个稳定的排序算法呢？想了很久，忽然觉得问题没有这么复杂。

快速排序的核心过程是 Partition 函数，这个函数的作用是选择一个基准（pivot），使比之小（或等于）的数都在结果列表中处于其左边，比之大的数都在结果列表中处于其右边。我们发现，快速排序不稳定性的根源就来自于此。比如这个例子：
【1，4，2(黄)，3，6，2(绿)，5，2(蓝)】
列表中出现了三个2，那么我们如果选择中间的2（绿色）作为基准，则第一次Partition结束的列表成为
【1，2，2，2，4，3，6，5】
或者
【1，2，2，2，4，3，6，5】
可见，黄色和蓝色的2都被分在了绿色2的左边或者右边，明显三个2的相对顺序变化了。

结论：如果一个数字在待排序的列表中出现三次或以上，而这个数字在列表中出现的（非首次和末次）一次被选为基准（pivot），则结果肯定是不稳定的。

因此，通过更改比较时>=符号为>符号是没有意义的，因为不稳定的根源在于基准的选取。
如果要得到稳定的快速排序，必须在选取基准时避免这种情况，而且在交换元素时要小心不要打乱相同元素的相对顺序。
一种可选的做法是先记录相同元素的相对位置，再进行排序。这需要额外的空间开销。

* 归并排序
分治的思想, 划分到两个子数组均只有一个元素, 再比较, 再merge 两个有序的子序列, 假设归并n 元素的时间复杂度是T(n) , 则T(n) = 2T(n/2)+O(n)(为合并两个元素个数为n/2 的有序子序列的时间复杂度)
, 经过推导得: T(n) =O(nlgn)

> 时间复杂度分析: 
> In the _worst_ case, merge sort does about 39% fewer comparisons than [quicksort](https://en.wikipedia.org/wiki/Quicksort "Quicksort") does in the _average_ case. In terms of moves, merge sort's worst case complexity is [O](https://en.wikipedia.org/wiki/Big_O_notation "Big O notation")(_n_ log _n_)—the same complexity as quicksort's best case, and merge sort's best case takes about half as many iterations as the worst case

如图: 

其实时间复杂度就是merge 两个有序数组时, merge 的次数, 就是merge 的总次数乘每次数组的个数, 如上图, 就是这个树的总元素个数, 二叉树的树高是logN, 数组长度是N, 
总比较次数就是N*logN, 其实就是数每行的元素个数的累加总和. 

> 
```
class Merge {

    // 用于辅助合并有序数组
    private static int[] temp;

    public static void sort(int[] nums) {
        // 先给辅助数组开辟内存空间, 比在递归中开辟快
        temp = new int[nums.length];
        // 排序整个数组（原地修改）
        sort(nums, 0, nums.length - 1);
    }

    // 定义：将子数组 nums[lo..hi] 进行排序
    private static void sort(int[] nums, int lo, int hi) {
        if (lo == hi) {
            // 单个元素不用排序
            return;
        }
        // 这样写是为了防止溢出，效果等同于 (hi + lo) / 2
        int mid = lo + (hi - lo) / 2;
        // 先对左半部分数组 nums[lo..mid] 排序
        sort(nums, lo, mid);
        // 再对右半部分数组 nums[mid+1..hi] 排序
        sort(nums, mid + 1, hi);
        // 将两部分有序数组合并成一个有序数组
        merge(nums, lo, mid, hi);
    }

    // 将 nums[lo..mid] 和 nums[mid+1..hi] 这两个有序数组合并成一个有序数组
    private static void merge(int[] nums, int lo, int mid, int hi) {
        // 先把 nums[lo..hi] 复制到辅助数组中
        // 以便合并后的结果能够直接存入 nums
        for (int i = lo; i <= hi; i++) {
            temp[i] = nums[i];
        }

        // 数组双指针技巧，合并两个有序数组
        int i = lo, j = mid + 1;
        for (int p = lo; p <= hi; p++) {
            if (i == mid + 1) {
                // 左半边数组已全部被合并
                nums[p] = temp[j++];
            } else if (j == hi + 1) {
                // 右半边数组已全部被合并
                nums[p] = temp[i++];
            } else if (temp[i] > temp[j]) {
                nums[p] = temp[j++];
            } else {
                nums[p] = temp[i++];
            }
        }
    }
}

```

## 快速排序
选一个基准数, 小于基准数的放到右边, 大于的放到左边, 并一直递归到数组只有一个元素, 再返回, 平均时间复杂度: O(nlgn), 

> 最好时间复杂度

数组已经按升序排好序, pivot 为中间元素, 则第一次遍历需要n 次比较, 然后均分成n/2 和 n/2 的两组进行递归, 满足O(n) = 2O(n/2)+ O(n), 则最好时间复杂度为O(nlgn)

> 最坏时间复杂度 O(n 的平方)

数组元素为降序, 当需要排成升序的时候. 或者元素都相等的时候(没有使用三分法). 为O(n的平方)
详见 https://en.wikipedia.org/wiki/Quicksort  " Worst-case analysis"

例如 4, 3, 2, 1, 之前为降序需要变为升序, pivot 选了最左边的元素, 
```
4 3 2 1

3 2 1 4

2 1 3 4

1 2 3 4
需要经历这个过程, 就是 O(n 的平方)
```
所以需要加入随机选择 pivot , 避免退化成 O(n 的平方),

> 平均时间复杂度待研究, 需要看算法推导


> 版本一，加入随机，避免交换次数过多，退化为O（n的平方）


```
class Solution {
    Random r = new Random();
    public void quickSort(int[] nums) {
        qs(nums, 0, nums.length - 1);
    }

    void qs(int [] nums, int s, int e){
        int p = partition(nums, s, e);
        qs(nums, p+1, e);
        qs(nums, s, p-1);
    }

    int partition(int [] nums ,int s, int e){
        int ran = r.nextInt(e - s + 1) + s; // s <= ran <= e
        swap(nums, ran, e);
        int pivot = nums[e];
        int j = s;
        for(int i = s; i < e; i++){ // 因为pivot 是 e, 所以 i < e
            if(nums[i] <= pivot){
                swap(nums, i, j++);
            }
        }
        swap(nums, j, e); // 此时 nums[j] > pivot
        return j;
    }

    void swap(int [] nums, int a, int b){
        int tmp = nums[a];
        nums[a] = nums[b];
        nums[b] = tmp;
    }
}
```

> 版本二，不随机
```
// 思想: 把小于pivot 的数移到前面, 大于pivot 的数移到后面, 然后用一个指针指向第一个大于pivot 的数, 最后交换这个指针和末尾指针.
static void QSRecu(int[] arr, int start, int end) {  
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
  
    QSRecu(arr, start, l - 1);  
    QSRecu(arr, l + 1, end);  
}


public void quickSort(int [] arr){
    QSRecu(arr, 0, arr.length - 1);
}
```

> 版本三, 三分法, 针对有元素相等的情况下, 

```
// 三分法, 用三个指针, 左边指针-1为小于pivot的index, 右边指针+1 位大于pivot 的index  , 第三个指针是比较指针 pivot 为最左边的start
void QSRecu(int[] arr, int start, int end) {  
    if (start >= end) return;  
    int pivot = arr[end];  
    int left = start + 1;  
    int s = start;  
    int right= end;  
    while (left <= right) {  
        if (arr[left] < pivot) swap(arr, s++, left++);  
        else if (arr[left] > pivot) swap(arr, left,right--);  
        else left++;  
    }  
    QSRecu(arr, start, s - 1);  
    QSRecu(arr, right+ 1, end);  
}

public void quickSort(int [] arr){
    QSRecu(arr, 0, arr.length - 1);
}
```


### topK 问题, 从无序数组中找到 topK 元素
以下示例是找最小的 topK, 最大则相反

* 方法 1 , 堆排序, 时间复杂度 O(nlogk), 空间复杂度 O(k)
```
public int[] getLeastNumbers(int[] arr, int k) {
    if (k == 0) {
        return new int[0];
    }
    // 使用一个最大堆（大顶堆）
    // Java 的 PriorityQueue 默认是小顶堆，添加 comparator 参数使其变成最大堆, 堆顶为最大元素
    Queue<Integer> heap = new PriorityQueue<>(k, (i1, i2) -> Integer.compare(i2, i1));

    for (int e : arr) {
        // 当前数字小于堆顶元素才会入堆
        if (heap.isEmpty() || heap.size() < k || e < heap.peek()) {
            heap.offer(e);
        }
        if (heap.size() > k) {
            heap.poll(); // 删除堆顶最大元素
        }
    }

    // 将堆中的元素存入数组
    int[] res = new int[heap.size()];
    int j = 0;
    for (int e : heap) {
        res[j++] = e;
    }
    return res;
}

```

* 方法 2 快速排序
时间复杂度 O(n), 
最坏时间复杂度 O(n 的平方), 
例如 4, 3, 2, 1 找最小的一个元素, pivot 选了最左边的元素, 
```
4 3 2 1

3 2 1 4

2 1 3 4

1 2 3 4
需要经历这个过程, 就是 O(n 的平方)
```
所以需要加入随机避免退化成 O(n 的平方), 以下是代码

```
class Solution {
    Random r = new Random();
    // quick sort to find top K, 加入随机避免退化成 O(n 的平方)
    public int findKthLargest(int[] nums, int k) {
        int t = nums.length - k;
        return qs(nums, 0, nums.length - 1, t);
    }

    int qs(int [] nums, int s, int e, int t){
        int p = partition(nums, s, e);// 返回下标, 下标为 k 即前面有了 k 个数.
        if(p == t){
            return nums[p];
        }else if(t > p){
            return qs(nums, p+1, e, t);
        }
        return qs(nums, s, p-1, t);
    }

    int partition(int [] nums ,int s, int e){
        int ran = r.nextInt(e - s + 1) + s; // s <= ran <= e
        swap(nums, ran, e);
        int pivot = nums[e];
        int j = s;
        for(int i = s; i < e; i++){ // 因为pivot 是 e, 所以 i < e
            if(nums[i] <= pivot){
                swap(nums, i, j++);
            }
        }
        swap(nums, j, e); // 此时 nums[j] > pivot
        return j;
    }

    void swap(int [] nums, int a, int b){
        int tmp = nums[a];
        nums[a] = nums[b];
        nums[b] = tmp;
    }
}
```


## 冒泡排序
每完全冒泡一次, 就把一个最大或最小的元素放到了数组的一端, 最坏情况需要O(n*n) 交换, 而插入排序最坏情况只需要O(n) 交换, 因为新插入的元素是可以跟有序子序列的每个元素进行比较, 而冒泡排序只能跟相邻元素进行比较
```
static void popS(int [] arr){  
     int len = arr.length;  
     for(int i=len-1;i>=1;i--){  
         for(int j=0;j+1<=i;j++){  
             if(arr[j] >arr[j+1]){  
                 int swap = arr[j];  
                 arr[j]=arr[j+1];  
                 arr[j+1]=temp;  
             }  
         }  
     }  
 }
```

## 插入排序
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
## 选择排序
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
## 堆排序
如何记忆, 想象每次都是构建最大堆, 那么二叉树如何构建最大元素是根节点的呢, 就是从非叶子节点开始逐步冒泡呗, 把最大元素冒泡到根节点就完成了一次构建最大堆, 然后把根节点元素放到数组末尾, 再从数组末尾-1 的位置再次构建最大堆
因为一个数组可以以完全二叉树的形式表示, 左子树下标是根的 2n+1, 右子树下标是根的 2n+2, 那如何转换为最大堆呢, 最大元素是根元素; 
1. 找到第一个下标最小的非叶子节点(数组长度除以2), 
2. 然后跟叶子节点比较, 把最大的元素交换并放到根节点, 
3. 再依次递减遍历所有节点, 并每次交换完之后再次执行步骤2, 判断叶子节点是否符合最大堆 

是最好的增量排序方式, 比二叉搜索树更加对cpu cache 友好, 
初始化建堆的时间复杂度O(n), 整个算法的时间复杂度: O(nlgn)+O(n),
建堆的有两种方式, 一种是bottom-up 就是下面这种, 时间复杂度O(n), 另一种是 top-down , 时间复杂度是O(nlogn), 如何实现呢? 为什么会有这个差异? 
```

  static int[] heapSortAsc(int[] arr) {
    // 第一步, 建堆的时间复杂度: O(n)
    for (int i = arr.length / 2; i >= 0; i--) {
      buildMaxHeap(arr, i, arr.length - 1);
    }

    // 第二步, O(nlog(n))
    for (int i = arr.length - 1; i >= 1; i--) {
      int tmp = arr[0];
      arr[0] = arr[i];
      arr[i] = tmp;
      buildMaxHeap(arr,   0, i - 1);
    }
    return arr;
  }

// 第三步: 堆调整的时间复杂度O(logn), 为树的高度, 因为第二步每次都是最大元素和最末尾元素交换, 然后末尾元素是较小的, 都要交换到树底, 所以复杂度为树的高度. 
  static void buildMaxHeap(int[] arr, int node, int heapSize) {
    int l = node * 2 + 1;
    int r = node * 2 + 2;
    if (l > heapSize) return;
    int max = l;
    if (r <= heapSize && arr[r] > arr[l]) {
      max = r;
    }
    if (arr[max] > arr[node]) {
      int tmp = arr[node];
      arr[node] = arr[max];
      arr[max] = tmp;
    }
    buildMaxHeap(arr, max, heapSize);
  }

```


## 希尔排序
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
## 桶排序
将元素放到几个子数组里面, 按照跟min 元素的差值来划分, 并在每个子数组中进行插入排序. 最坏时间复杂度O(n*n), 平均时间复杂度O(n), 见英文维基百科
# 二叉搜索树
插入是从顶向下, 递归的形式, 比节点小, 则比左儿子, 比节点大, 则比右儿子,  新插入的值总是落在叶子节点.

# 布隆过滤器
用途在于数据量非常庞大的时候, 判断数据是否在集合中, 减少数据集的空间占用, 用一个bit 数组来保存, 假设有n个hash 函数, 新来的一个元素a, 
插入: 将a 通过n 个hash 函数算出的结果分别对应bit 数组上的n 个bit 位, 分别置一; 
查询: 元素通过hash 函数算出的结果, 对应的bit 位上都是一, 则有可能在集合中, 因为n 个hash 都有可能冲突; 如果有一个bit 位为零, 则一定不存在集合中. 












# 经典题目
## 排序

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

## 字符串

### 如果字符串已经限定了是ASCII, 则可以用一个256 长度的int 数组来同等表示, 而且可以根据int 值的大小来排序, 排序的时间复杂度是O(n) .
### 滑动窗口的思想

## 数组
### 滑动窗口的思想
### 双指针逼近, 要有越来越靠近的趋势
## 查找

### 万变不离其中, 最快的查找就是二分查找, 时间复杂度O(lgn), 其次是线性时间复杂度.


## 二叉树

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

## 数字题

因为所有的数字都是二进制, 可以用二进制中独有的运算, 位运算来达到四两拨千斤的效果, 

## 位运算

* 负数如何用二进制表示

将绝对值用二进制表示, 然后取反, 然后加1, 最左边第一位为符号位, 若是负数则为1, 正数则为0; 
负数还原为正数: 第一位为符号位, 若为负数, 则先减一, 再取反. 

* 应用
 1. 异或, a^b, 不同的数异或结果为1 , 得出的二进制位哪位为1, 则哪位不同. 用于找只出现一次或两个只出现一次的数字 
 2. 高低位交换, 先左移, 再右移, 最后再或 





## 链表
```
// 反转链表
ListNode res = null;

void reverseLinkedList(ListNode node){
    help(node);
    return res;
}

ListNode help(ListNode node){
    if(node.next == null){
        res = node;
        return node;
    }
    ListNode next = help(node.next);
    next.next = node;
    node.next = null;
    return node;
}
```

```
// 反转链表的一部分
ListNode tail = null;
ListNode recuHead = null;
void reverseLinkedList(ListNode node, int left, int right){
    int i=1;
    ListNode begin = null;
    ListNode head = node;
    while(i++ < left){
        begin = node;
        node = node.next;
    }
    ListNode recuTail = help(node, i, right );
    recuTail.next = tail;
    begin.next = recuHead;
    return head;
}

ListNode help(ListNode node, int index, int end){
    if(index == end){
        recuHead = node;
        tail = node.next;
        return node;
    }
    ListNode next = help(node.next);
    next.next = node;
    node.next = null;
    return node;
}

```
```
// 反转链表前 N 个节点
ListNode successor = null;
ListNode reverseN(ListNode head, int n){
    if(n == 1){
        successor = head.next;
        return head;
    }
    ListNode last = reverseN(head.next, n-1);
    head.next.next =head;
    head.next = successor;
    return last;
}

```
**第一次做出递归hard 题, 感觉真的懂如何用递归了, 不需要辅助函数不需要外部变量, 爽!**
```
// https://leetcode.com/problems/reverse-nodes-in-k-group/
/** 
首先，前文 学习数据结构的框架思维 提到过，链表是一种兼具递归和迭代性质的数据结构，认真思考一下可以发现这个问题具有递归性质。

什么叫递归性质？直接上图理解，比如说我们对这个链表调用 reverseKGroup(head, 2)，即以 2 个节点为一组反转链表：


如果我设法把前 2 个节点反转，那么后面的那些节点怎么处理？后面的这些节点也是一条链表，而且规模（长度）比原来这条链表小，这就叫子问题。 **/
 ListNode reverseKGroup(ListNode head, int k){
     if(k == 1){
         return head;
     }
    int count = k;
    ListNode c = head;
    ListNode res = null;
    while(head != null && k > 0){
        res = head;
       head = head.next;
       k--;
    }
    if(k != 0){
        return c;
    }
    ListNode nextK = reverseKGroup(head, count);
    ListNode prevC = c;
    ListNode nn = null;
    ListNode next = c.next;
    while(count-- > 1 && next != null){
        nn = next.next;
        next.next = c;
        c = next;
        next = nn;
    }
    prevC.next = nextK;
    return res;
}

```
### 链表常见思路
* 快慢指针法, 一个一次一步, 一个一次两步找链表的中间节点. 
* 先用一个指针跑k 步, 再用第二个指针从头开始跑, 那么第二个指针跑的步就是第一个指针到末尾节点跑的步.
* 两步和一步指针判断有没有环, 如果有, 再相交时, 再用一步指针, 从头跑, 再次相交的位置就是环的起点.
* FakeHead 把应该返回的答案放到FakeHead 的next 节点, 与后续代码模式就匹配了, 不需要写额外的if else.
* 判断链表有没有相交, 可以用两个指针, 走完A 链表继续走B 链表, 如果相交最后肯定汇合到一个点. 


## 递归
* 如何判断题目是否能用递归呢
看题目是否能找出子问题或者类似问题(只是参数变化), 以及子问题和父问题的关系, 最后递归结束时为一般情况;
并且写递归时不要脑子里想着所有压栈的情况, 记不过来的, 只需要确定递归函数的定义和返回值, 根据第二次递归返回和第一次递归(第几次意思为第几次进入递归函数), 以及最后一次递归和倒数第二次递归来写代码. 
例如 reverse-nodes-in-k-group 这题, 真正的递归是多数不需要外部变量或者辅助函数的, 当然不需要纠结这个. 

* 什么是递归
递归可以想象成函数调用不断压栈, 直到满足**结束条件**, 最后一个调用首先返回. 
在递归调用之前的语句会正序执行, 再递归调用之后的语句会逆序执行(尾递归), 空间复杂度是递归树的深度也是return 的次数, 时间复杂度是递归基本函数调用的次数.
* 
> 注意事项
1. 注意递归中, 传递的参数的类型, 基本类型例如int, char 等, 在函数栈的嵌套中拷贝一个新的值到下一个函数调用, 每个函数栈的变量的改变不影响其他函数栈; 但是如果是对象类型, 则传递的是对象的引用, 多个函数栈中会改变对象的值; 但是如果传递的是新的字符串, 则不会改变
2. 灵活使用递归中的全局变量和参数, 能让递归更易理解. 
 

## 回溯思想(DFS 深度优先搜索)
类似深度遍历一个多叉树

https://leetcode.com/problems/permutations/discuss/18239/A-general-approach-to-backtracking-questions-in-Java-(Subsets-Permutations-Combination-Sum-Palindrome-Partioning)
常作为暴力破解, 穷举的一种优化, 类似于走迷宫,  走错了回头, 递归形式, 类似遍历一个多叉树, 与二叉搜索树的递归模式类似. 

代码框架
```
result = []
路径理解为结果集
void backtrack(路径, 选择列表):
    if 满足结束条件:
        result.add(路径)
        return
    
    for 选择 in 选择列表:
        做选择
        如果需要去重: if(j>i && a[j] == a[j-1]) continue;
        backtrack(路径, 选择列表)
        撤销选择 


```
## BFS
与dfs 的区别, 空间复杂度高, 但是对于特定任务时间复杂度低.
BFS 可以找到最短距离，但是空间复杂度高，而 DFS 的空间复杂度较低。

还是拿刚才我们处理二叉树问题的例子，假设给你的这个二叉树是满二叉树，节点数为 N，对于 DFS 算法来说，空间复杂度无非就是递归堆栈，最坏情况下顶多就是树的高度，也就是 O(logN)。

但是你想想 BFS 算法，队列中每次都会储存着二叉树一层的节点，这样的话最坏情况下空间复杂度应该是树的最底层节点的数量，也就是 N/2，用 Big O 表示的话也就是 O(N)。

由此观之，BFS 还是有代价的，一般来说在找最短路径的时候使用 BFS，其他时候还是 DFS 使用得多一些（主要是递归代码好写）


* 代码框架
```
// 计算从起点 start 到终点 target 的最近距离
int BFS(Node start, Node target) {
    Queue<Node> q; // 核心数据结构
    Set<Node> visited; // 避免走回头路
    
    q.offer(start); // 将起点加入队列
    visited.add(start);
    int step = 0; // 记录扩散的步数

    while (q not empty) {
        int sz = q.size();
        /* 将当前队列中的所有节点向四周扩散 */
        for (int i = 0; i < sz; i++) {
            Node cur = q.poll();
            /* 划重点：这里判断是否到达终点 */
            if (cur is target)
                return step;
            /* 将 cur 的相邻节点加入队列 */
            for (Node x : cur.adj()) {
                if (x not in visited) {
                    q.offer(x);
                    visited.add(x);
                }
            }
        }
        /* 划重点：更新步数在这里 */
        step++;
    }
}
```
例题
```
// 二叉树最小高度
int minDepth(TreeNode root) {
    if (root == null) return 0;
    Queue<TreeNode> q = new LinkedList<>();
    q.offer(root);
    // root 本身就是一层，depth 初始化为 1
    int depth = 1;
    
    while (!q.isEmpty()) {
        int sz = q.size();
        /* 将当前队列中的所有节点向四周扩散 */
        for (int i = 0; i < sz; i++) {
            TreeNode cur = q.poll();
            /* 判断是否到达终点 */
            if (cur.left == null && cur.right == null) 
                return depth;
            /* 将 cur 的相邻节点加入队列 */
            if (cur.left != null)
                q.offer(cur.left);
            if (cur.right != null) 
                q.offer(cur.right);
        }
        /* 这里增加步数 */
        depth++;
    }
    return depth;
}
```
## DFS vs BFS
```
// https://leetcode.cn/problems/number-of-provinces , 这题有BFS 和DFS 两种解法， DFS 就是深度， 一直往下走， BFS 是把相邻的城市先走完，按照顺序加到队列里。
    public int findCircleNumDfs(int[][] isConnected) {
        int city = isConnected.length;
        boolean [] visit = new boolean [city];
        int res = 0;
        for(int i = 0; i < city; i++){
            
            if(!visit[i]){
                res++;
                visit[i] = true;
                dfs(i, isConnected, visit);
            }
        }
        return res;
    }

    void dfs(int i, int [][] isConnected, boolean [] visit){
        int [] path = isConnected[i];
        for(int j = 0; j < path.length; j++){
            if(path[j] == 1 && !visit[j]){
                visit[j] = true;
                dfs(j, isConnected, visit);
            }
        }
    }

     public int findCircleNumBfs(int[][] isConnected) {
         // 1818
         int city = isConnected.length;
        boolean [] visit = new boolean [city];
        int res = 0;
        Queue<Integer> q = new LinkedList<>();
        for(int i = 0; i < city; i++){
            if(!visit[i]){
              q.add(i);
              res++;
              while(!q.isEmpty()){
                  int c = q.poll();
                  visit[c] = true;
                  for(int j = 0; j < city; j++){
                      if(isConnected[c][j] == 1 && !visit[j]){
                          q.add(j);
                      }
                  }
              }
            }
        }
        return res;
     }
```
# 跳表
参考资料: https://www.jianshu.com/p/9d8296562806
假设原始链表有 n 个元素, 那么随机抽 n /2 个元素作为一层索引, 二层是 n/4, 三层是 n/8, 以此类推, 1/2 则为跳表系数, 最高层一般是两个元素, 那么索引高度假设为h, 不考虑原始链表, 那么第h 层元素个数为2, 可得 n/(2的h次方) = 2, 那么 h = log2 n - 1;
</br>
* 查询时间复杂度
  
查询的时间复杂度等于跳表的高度乘每层比较的次数(常数), 忽略常数之后约等于 O(lgn), n 为元素个数, 近似为二分查找, 
每层高度比较的次数如何计算, 假设跳表系数为1/2, 那么从最高层开始比较, 一开始比较次数为2, 第二层为1, 以此类推, 可以想象比较次数是跟跳表系数有关, 是一个常数, 跳表系数越小, 需要的比较次数就越多, 但是空间占用也越少, 
</br> 
* 插入时间复杂度

分两个方面, 一是计算 randomLevel(), 是 O(1), 二是计算得到 level=3, 三层索引, 那么三层以下的每层包括原始链表都需要插入该元素, 那么最坏时间复杂度就是需要每层都插入, 为高度 O(log n)

* 删除时间复杂度

首先需要找到元素, 为查询时间复杂度 O(log n), 然后开始删除元素, 最坏情况是每层都要删除该元素, 每个链表删除元素的时间复杂度是 O(1), 那么删除的复杂度就是树的高度 O(log n), 
总复杂度就是查询加删除, 为 2*O(log n), 约等于 O(log n).
* 空间复杂度

假设跳表系数为 1/2, 那么每层索引的元素个数就是 n/2, n/4 ... 2, 等比数列累加和为 n. 但是每级索引只保留了 key, val 只保存在最底层数据中. 那么索引相对于实际数据占比可以说很小. 
空间复杂度是 O(n), 

* randomLevel
![image](https://github.com/Knight-Wu/articles/assets/20329409/fa82dd9b-988b-4666-a6a5-cbb248959922)

假设我们在插入元素 6 的时候，randomLevel() 方法返回 1，则我们不会为 6 建立索引。插入 7 的时候，randomLevel() 方法返回3 ，所以我们需要为元素 7 建立二级索引。这里我们发现了一个特点：当建立二级索引的时候，同时也会建立一级索引；当建立三级索引时，同时也会建立一级、二级索引。所以，一级索引中元素的个数等于 [ 原始链表元素个数 ] * [ randomLevel() 方法返回值 > 1 的概率 ]。因为 randomLevel() 方法返回值 > 1就会建索引，凡是建索引，无论几级索引必然有一级索引，所以一级索引中元素个数占原始数据个数的比率为 randomLevel() 方法返回值 > 1 的概率。那 randomLevel() 方法返回值 > 1 的概率是多少呢？因为 randomLevel() 方法随机生成 1~MAX_LEVEL 的数字，且 randomLevel() 方法返回值 1 的概率为 1/2，则 randomLevel() 方法返回值 > 1 的概率为 1 - 1/2 = 1/2。即通过上述流程实现了一级索引中元素个数占原始数据个数的 1/2。
</br>
同理，当 randomLevel() 方法返回值 > 2 时，会建立二级或二级以上索引，都会在二级索引中增加元素，因此二级索引中元素个数占原始数据的比率为 randomLevel() 方法返回值 > 2 的概率。 randomLevel() 方法返回值 > 2 的概率为 1 减去 randomLevel() = 1 或 =2 的概率，即 1 - 1/2 - 1/4 = 1/4。OK，达到了我们设计的目标：二级索引中元素个数占原始数据的 1/4。

以此类推，可以得出，遵守以下两个条件：

randomLevel() 方法，随机生成 1~MAX_LEVEL 之间的数（MAX_LEVEL表示索引的最高层数），且有 1/2的概率返回 1、1/4的概率返回 2、1/8的概率返回 3 ...
randomLevel() 方法返回 1 不建索引、返回2建一级索引、返回 3 建二级索引、返回 4 建三级索引 ...
就可以满足我们想要的结果，即：一级索引中元素个数应该占原始数据的 1/2，二级索引中元素个数占原始数据的 1/4，三级索引中元素个数占原始数据的 1/8 ，依次类推，一直到最顶层索引。

但是问题又来了，怎么设计这么一个 randomLevel() 方法呢？直接撸代码：
```
// 该 randomLevel 方法会随机生成 1~MAX_LEVEL 之间的数，且 ：
//        1/2 的概率返回 1
//        1/4 的概率返回 2
//        1/8 的概率返回 3 以此类推
private int randomLevel() {
  int level = 1;
  // 当 level < MAX_LEVEL，且随机数小于设定的晋升概率时，level + 1
  while (Math.random() < SKIPLIST_P && level < MAX_LEVEL)
    level += 1;
  return level;
}
```
上述代码可以实现我们的功能，而且，我们的案例中晋升概率 SKIPLIST_P 设置的 1/2，即：每两个结点抽出一个结点作为上一级索引的结点。如果我们想节省空间利用率，可以适当的降低代码中的 SKIPLIST_P，从而减少索引元素个数，Redis 的 zset 中 SKIPLIST_P 设定的 0.25。
下图所示，是Redis t_zset.c 中 zslRandomLevel 函数的实现：
![image](https://github.com/Knight-Wu/articles/assets/20329409/72eafd0c-5bc5-4f74-b04d-526a7062dff7)

Redis 源码中 (random()&0xFFFF) < (ZSKIPLIST_P * 0xFFFF) 在功能上等价于我代码中的 Math.random() < SKIPLIST_P ，只不过 Redis 作者 antirez 使用位运算来提高浮点数比较的效率。

* 为什么Redis选择使用跳表而不是红黑树来实现有序集合？
Redis 中的有序集合(zset) 支持的操作：

1. 插入一个元素

2. 删除一个元素

3. 查找一个元素

4. 有序输出所有元素

5. 按照范围区间查找元素

其中，前四个操作红黑树也可以完成，且时间复杂度跟跳表是一样的。但是，按照区间来查找数据这个操作，红黑树的效率没有跳表高。按照区间查找数据时，跳表可以做到 O(logn) 的时间复杂度定位区间的起点，然后在原始链表中顺序往后遍历就可以了，非常高效


* 使用场景

适用于 LSM 类存储引擎(Hbase, levelDB)做 memStore 的数据结构, 以及 redis zset , 因为插入, 查询, 删除的时间复杂度都是 O(lgn) , 而且底层链表又是有序的, 可以直接持久化做 SSTable, 

# 资源
* 算法第四版
https://algs4.cs.princeton.edu/
* coursera
https://www.coursera.org/learn/algorithms-part1/

# 待解决问题
* 回文字符串
https://leetcode.com/problems/rotate-string/solution/
