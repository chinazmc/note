#review/first_101_算法 

```text
//剑指 Offer 42. 连续子数组的最大和  
//似乎遇到数组求最大值之类的问题就可以采用数组前缀和的方式来做  
/**  
 * 输入一个整型数组，数组中的一个或连续多个整数组成一个子数组。求所有子数组的和的最大值。  
 *  
 * 要求时间复杂度为O(n)。  
 *  
 * 示例1:  
 * * 输入: nums = [-2,1,-3,4,-1,2,1,-5,4]  
 * 输出: 6  
 * 解释: 连续子数组 [4,-1,2,1] 的和最大，为 6。  
 * 提示：  
 *  
 * 1 <= arr.length <= 10^5 * -100 <= arr[i] <= 100 * */
```
?
```java
public static int maxSubArray(int[] nums) {  
    int res=nums[0];  
 int n=nums.length;  
 for (int i=1;i<n;i++){  
        nums[i]+=Math.max(nums[i-1],0);  
		 res=Math.max(res,nums[i]);  
 }  
    return res;  
}
```
<!--SR:!2023-10-15,79,210-->

```text
/**  
 * 剑指 Offer 10- I. 斐波那契数列  
 * 写一个函数，输入 n ，求斐波那契（Fibonacci）数列的第 n 项（即 F(N)）。斐波那契数列的定义如下：  
 *  
 * F(0) = 0,   F(1) = 1 * F(N) = F(N - 1) + F(N - 2), 其中 N > 1.  
 * 斐波那契数列由 0 和 1 开始，之后的斐波那契数就是由之前的两数相加而得出。  
 *  
 * 答案需要取模 1e9+7（1000000007），如计算初始结果为：1000000008，请返回 1。  
 *  
 * 示例 1：  
 * 输入：n = 2  
 * 输出：1  
 * * 示例 2：  
 * 输入：n = 5  
 * 输出：5  
 * * */
```
?
```java
/**  
 * 空间复杂度优化：  
 * 若新建长度为 n 的 dp 列表，则空间复杂度为 O(N) 。  
 *  
 * 由于 dp 列表第 i 项只与第 i−1 和第 i−2 项有关，因此只需要初始化三个整形变量 sum, a, b ，利用辅助变量 sum 使 a, b 两数字交替前进即可 （具体实现见代码） 。  
 * 节省了 dp 列表空间，因此空间复杂度降至 O(1) 。  
 *  
 * */ 
 public static int myFib(int n){  
        Map<Integer,Integer> map=new HashMap<>();  
		map.put(0,0);  
		map.put(1,1);  
		return test(n-1,map)+test(n-2,map);  
 }  
    public static int test(int n,Map<Integer,Integer> map){  
        if (map.containsKey(n)) return map.get(n);  
		return test(n-1,map)+test(n-2,map);  
 }
/**  
 * 循环求余法：  
 * 大数越界： 随着 n 增大, f(n) 会超过 Int32 甚至 Int64 的取值范围，导致最终的返回值错误。  
 *  
 * 求余运算规则： 设正整数 x, y, p，求余符号为 ⊙ ，则有 (x + y) ⊙ p = (x ⊙ p + y ⊙ p) ⊙ p 。  
 * 解析： 根据以上规则，可推出 f(n) ⊙ p = [f(n-1) ⊙ p + f(n-2) ⊙ p] ⊙ p，从而可以在循环过程中每次计算 sum = (a + b) ⊙ 1000000007,此操作与最终返回前取余等价。  
 *  
 * 复杂度分析：  
 * 时间复杂度 O(N) ： 计算 f(n) 需循环 n 次，每轮循环内计算操作使用 O(1) 。  
 * 空间复杂度 O(1) ： 几个标志变量使用常数大小的额外空间。  
 *  
 * */
 public static int fib(int n){  
	 int a=0,b=1;  
	 int sum=0;  
	 for (int i=0;i<n;i++){  
	    sum=(a+b)%1000000007;  
		 a=b;  
		 b=sum;  
	 }  
    return a;  
}
```
<!--SR:!2023-10-04,95,250-->

```java
[1137. 第 N 个泰波那契数]

泰波那契序列 Tn 定义如下： 

T0 = 0, T1 = 1, T2 = 1, 且在 n >= 0 的条件下 Tn+3 = Tn + Tn+1 + Tn+2

给你整数 `n`，请返回第 n 个泰波那契数 Tn 的值。

示例 1：

输入：n = 4
输出：4
解释：
T_3 = 0 + 1 + 1 = 2
T_4 = 1 + 1 + 2 = 4

示例 2：

输入：n = 25
输出：1389537
提示：

0 <= n <= 37
答案保证是一个 32 位整数，即 answer <= 2^31 - 1。
```
?
```java
class Solution {
    int[] cache = new int[40];
    public int tribonacci(int n) {
        if (n == 0) return 0;
        if (n == 1 || n == 2) return 1;
        if (cache[n] != 0) return cache[n];
        cache[n] = tribonacci(n - 1) + tribonacci(n - 2) + tribonacci(n - 3); 
        return cache[n];
    }
}
时间复杂度和空间复杂度都是O(n)
```
<!--SR:!2023-11-25,109,210-->


```java
  
//LCP 07. 传递信息  
//        小朋友 A 在和 ta 的小伙伴们玩传信息游戏，游戏规则如下：  
//  
//        有 n 名玩家，所有玩家编号分别为 0 ～ n-1，其中小朋友 A 的编号为 0
//        每个玩家都有固定的若干个可传信息的其他玩家（也可能没有）。传信息的关系是单向的（比如 A 可以向 B 传信息，但 B 不能向 A 传信息）。  
//        每轮信息必须需要传递给另一个人，且信息可重复经过同一个人  
//        给定总玩家数 n，以及按 [玩家编号,对应可传递玩家编号] 关系组成的二维数组 relation。返回信息从小 A (编号 0 ) 经过 k 轮传递到编号为 n-1 的小伙伴处的方案数；若不能到达，返回 0。  
//  
//        示例 1：  
//   输入：n = 5, relation = [[0,2],[2,1],[3,4],[2,3],[1,4],[2,0],[0,4]], k = 3  
//        输出：3  
//  
//        解释：信息从小 A 编号 0 处开始，经 3 轮传递，到达编号 4。共有 3 种方案，分别是 0->2->0->4， 0->2->1->4， 0->2->3->4。  
//  
//        示例 2：  
//  
//        输入：n = 3, relation = [[0,2],[2,1]], k = 2  
//  
//        输出：0  
//  
//        解释：信息不能从小 A 处经过 2 轮传递到编号 2//  
//        限制：  
//  
//        2 <= n <= 10  
//        1 <= k <= 5  
//        1 <= relation.length <= 90, 且 relation[i].length == 2//        0 <= relation[i][0],relation[i][1] < n 且 relation[i][0] != relation[i][1]
```
?
```java
public class NumWays {  
//    BFS  
//    一个朴素的做法是使用 BFS 进行求解。  
//  
//    起始时，将起点入队，按照常规的 BFS 方式进行拓展，直到拓展完第 k 层。  
//  
//    然后统计队列中编号为 n-1 的节点的出现次数。  
//  
//    一些细节：为了方便找到某个点 i 所能到达的节点，我们需要先预处理出所有的边。数据量较少，直接使用 Map 套 Set 即可。  
//    时间复杂度：最多搜索 k 层，最坏情况下每层的每个节点都能拓展出 n−1 个节点，复杂度为 O(n^k)//    空间复杂度：最坏情况下，所有节点都相互连通。复杂度为 O(n^k + n^2)    public int numWays(int n, int[][] rs, int k) {  
        Map<Integer, Set<Integer>> map = new HashMap<>();  
        for (int[] r : rs) {  
            int a = r[0], b = r[1];  
            Set<Integer> s = map.getOrDefault(a, new HashSet<>());  
            s.add(b);  
            map.put(a, s);  
        }  
        Deque<Integer> d = new ArrayDeque<>();  
        d.addLast(0);  
        while (!d.isEmpty() && k-- > 0) {  
            int size = d.size();  
            while (size-- > 0) {  
                int poll = d.pollFirst();  
                Set<Integer> es = map.get(poll);  
                if (es == null) continue;  
                for (int next : es) {  
                    d.addLast(next);  
                }  
            }  
        }  
        int ans = 0;  
        while (!d.isEmpty()) {  
            if (d.pollFirst() == n - 1) ans++;  
        }  
        return ans;  
    }  
//    动态规划  
//    假设当前我们已经走了 i 步，所在位置为 j，那么剩余的 k−i 步，能否到达位置 n−1，仅取决于「剩余步数 k−i」和「边权关系 relation」，而与我是如何到达位置 i 无关。  
//    而对于方案数而言，如果已经走了 i 步，所在位置为 j，到达位置 n−1 的方案数仅取决于「剩余步数 i−k」、「边权关系 relation」和「花费 i 步到达位置 j 的方案数」。  
//    以上分析，归纳到边界「已经走了 0 步，所在位置为 0」同样成立。  
//    这是一个「无后效性」的问题，可以使用动态规划进行求解。  
//  
//    定义f[i][j] 为当前已经走了 i 步，所在位置为 j 的方案数。  
//  
//    那么f[k][n−1] 为最终答案，f[0][0]=1 为显而易见的初始化条件。  
//    时间复杂度：O(relation.length∗k)  
//    空间复杂度：O(n∗k)  
public int numWays1(int n, int[][] rs, int k) {  
    int[][] f = new int[k + 1][n];  
    f[0][0] = 1;  
    for (int i = 1; i <= k; i++) {  
        for (int[] r : rs) {  
            int a = r[0], b = r[1];  
            f[i][b] += f[i - 1][a];  
        }  
    }  
    return f[k][n - 1];  
}  
//    DFS  
//    同理，我们可以使用 DFS 进行求解。  
//  
//    在 DFS 过程中限制深度最多为 kk，然后检查所达节点为 n-1 的次数即可。  
//    时间复杂度：最多搜索 kk 层，最坏情况下每层的每个节点都能拓展出 n - 1n−1 个节点，复杂度为 O(n^k)//    空间复杂度：最坏情况下，所有节点都相互连通。忽略递归带来的额外空间消耗，复杂度为 O(n^2)Map<Integer, Set<Integer>> map = new HashMap<>();  
    int n, k, ans;  
    public int numWays2(int _n, int[][] rs, int _k) {  
        n = _n; k = _k;  
        for (int[] r : rs) {  
            int a = r[0], b = r[1];  
            Set<Integer> s = map.getOrDefault(a, new HashSet<>());  
            s.add(b);  
            map.put(a, s);  
        }  
        dfs(0, 0);  
        return ans;  
    }  
    void dfs(int u, int sum) {  
        if (sum == k) {  
            if (u == n - 1) ans++;  
            return;        }  
        Set<Integer> es = map.get(u);  
        if (es == null) return;  
        for (int next : es) {  
            dfs(next, sum + 1);  
        }  
    }  
  
}
```
<!--SR:!2023-08-06,9,170-->

```java
//45. 跳跃游戏 II
//        给你一个非负整数数组 nums ，你最初位于数组的第一个位置。  
//        数组中的每个元素代表你在该位置可以跳跃的最大长度。  
//        你的目标是使用最少的跳跃次数到达数组的最后一个位置。  
//        假设你总是可以到达数组的最后一个位置。  
//        示例 1://  
//        输入: nums = [2,3,1,1,4]  
//        输出: 2  
//        解释: 跳到最后一个位置的最小跳跃数是 2。  
//        从下标为 0 跳到下标为 1 的位置，跳 1 步，然后跳 3 步到达数组的最后一个位置。  
//        示例 2:
//        输入: nums = [2,3,0,1,4]  
//        输出: 2  
//        提示:  
//  
//        1 <= nums.length <= 104  
//        0 <= nums[i] <= 1000  
```
?
```java
public class Jump {  
//    正向查找可到达的最大位置  
//    方法一虽然直观，但是时间复杂度比较高，有没有办法降低时间复杂度呢？  
//    如果我们「贪心」地进行正向查找，每次找到可到达的最远位置，就可以在线性时间内得到最少的跳跃次数。  
//    例如，对于数组 [2,3,1,2,4,2,3]，初始位置是下标 0，从下标 0 出发，最远可到达下标 2。下标 0 可到达的位置中，下标 1 的值是 3，从下标 1 出发可以达到更远的位置，因此第一步到达下标 1。  
//    从下标 1 出发，最远可到达下标 4。下标 1 可到达的位置中，下标 4 的值是 4 ，从下标 4 出发可以达到更远的位置，因此第二步到达下标 4。  
//    在具体的实现中，我们维护当前能够到达的最大下标位置，记为边界。我们从左到右遍历数组，到达边界时，更新边界并将跳跃次数增加 1。  
//    在遍历数组时，我们不访问最后一个元素，这是因为在访问最后一个元素之前，我们的边界一定大于等于最后一个位置，否则就无法跳到最后一个位置了。如果访问最后一个元素，在边界正好为最后一个位置的情况下，我们会增加一次「不必要的跳跃次数」，因此我们不必访问最后一个元素。  
  
    public int jump(int[] nums) {  
        int length = nums.length;  
        int end = 0;  
        int maxPosition = 0;  
        int steps = 0;  
        for (int i = 0; i < length - 1; i++) {  
            maxPosition = Math.max(maxPosition, i + nums[i]);  
            if (i == end) {  
                end = maxPosition;  
                steps++;  
            }  
        }  
        return steps;  
    }  
  
//    复杂度分析  
//    时间复杂度：O(n)，其中 n 是数组长度。  
//    空间复杂度：O(1)。  
}
```
<!--SR:!2023-10-04,8,130-->

```java
//91. 解码方法  
//        一条包含字母 A-Z 的消息通过以下映射进行了 编码 ：  
//        'A' -> "1"  
//        'B' -> "2"  
//        ...  
//        'Z' -> "26"  
//        要 解码 已编码的消息，所有数字必须基于上述映射的方法，反向映射回字母（可能有多种方法）。例如，"11106" 可以映射为：  
//        "AAJF" ，将消息分组为 (1 1 10 6)//        "KJF" ，将消息分组为 (11 10 6)//        注意，消息不能分组为  (1 11 06) ，因为 "06" 不能映射为 "F" ，这是由于 "6" 和 "06" 在映射中并不等价。  
//        给你一个只含数字的 非空 字符串 s ，请计算并返回 解码 方法的 总数 。  
//        题目数据保证答案肯定是一个 32 位 的整数。  
//        示例 1：  
//  
//        输入：s = "12"  
//        输出：2  
//        解释：它可以解码为 "AB"（1 2）或者 "L"（12）。  
//        示例 2：  
//  
//        输入：s = "226"  
//        输出：3  
//        解释：它可以解码为 "BZ" (2 26), "VF" (22 6), 或者 "BBF" (2 2 6) 。  
//        示例 3：  
//  
//        输入：s = "0"  
//        输出：0  
//        解释：没有字符映射到以 0 开头的数字。  
//        含有 0 的有效映射是 'J' -> "10" 和 'T'-> "20" 。  
//        由于没有字符，因此没有有效的方法对此进行解码，因为所有数字都需要映射。  
//        提示：  
//  
//        1 <= s.length <= 100  
//        s 只包含数字，并且可能包含前导零。  

```
?
```java
public class NumDecodings {  
//    动态规划  
//    这其实是一道字符串类的动态规划题，不难发现对于字符串 s 的某个位置 i 而言，我们只关心「位置 i 自己能否形成独立 item 」和「位置 i 能够与上一位置（i-1）能否形成 item」，而不关心 i-1 之前的位置。  
//  
//    有了以上分析，我们可以从前往后处理字符串 s，使用一个数组记录以字符串 s 的每一位作为结尾的解码方案数。即定义 f[i]f[i] 为考虑前 ii 个字符的解码方案数。  
//  
//    对于字符串 s 的任意位置 i 而言，其存在三种情况：  
//  
//    只能由位置 i 的单独作为一个 item，设为 a，转移的前提是 a 的数值范围为 [1,9][1,9]，转移逻辑为 f[i] = f[i - 1]f[i]=f[i−1]。  
//    只能由位置 i 的与前一位置（i-1）共同作为一个 item，设为 b，转移的前提是 b 的数值范围为 [10,26][10,26]，转移逻辑为 f[i] = f[i - 2]f[i]=f[i−2]。  
//    位置 i 既能作为独立 item 也能与上一位置形成 item，转移逻辑为 f[i] = f[i - 1] + f[i - 2]f[i]=f[i−1]+f[i−2]。  
//  
//    其他细节：由于题目存在前导零，而前导零属于无效 item。可以进行特判，但个人习惯往字符串头部追加空格作为哨兵，  
//    追加空格既可以避免讨论前导零，也能使下标从 1 开始，简化 f[i-1] 等负数下标的判断。  
  
    public int numDecodings(String s) {  
        int n = s.length();  
        s = " " + s;  
        char[] cs = s.toCharArray();  
        int[] f = new int[n + 1];  
        f[0] = 1;  
        for (int i = 1; i <= n; i++) {  
            // a : 代表「当前位置」单独形成 item            // b : 代表「当前位置」与「前一位置」共同形成 item            int a = cs[i] - '0', b = (cs[i - 1] - '0') * 10 + (cs[i] - '0');  
            // 如果 a 属于有效值，那么 f[i] 可以由 f[i - 1] 转移过来  
            if (1 <= a && a <= 9) f[i] = f[i - 1];  
            // 如果 b 属于有效值，那么 f[i] 可以由 f[i - 2] 或者 f[i - 1] & f[i - 2] 转移过来  
            if (10 <= b && b <= 26) f[i] += f[i - 2];  
        }  
        return f[n];  
    }  
  
//    时间复杂度：共有 n 个状态需要被转移。复杂度为 O(n)。  
//    空间复杂度：O(n)。  
}
```
<!--SR:!2023-07-29,4,85-->

```java
//198. 打家劫舍  
//        你是一个专业的小偷，计划偷窃沿街的房屋。每间房内都藏有一定的现金，影响你偷窃的唯一制约因素就是相邻的房屋装有相互连通的防盗系统，如果两间相邻的房屋在同一晚上被小偷闯入，系统会自动报警。  
//  
//        给定一个代表每个房屋存放金额的非负整数数组，计算你 不触动警报装置的情况下 ，一夜之内能够偷窃到的最高金额。  
//  
//  
//  
//        示例 1：  
//  
//        输入：[1,2,3,1]  
//        输出：4  
//        解释：偷窃 1 号房屋 (金额 = 1) ，然后偷窃 3 号房屋 (金额 = 3)。  
//        偷窃到的最高金额 = 1 + 3 = 4 。  
//        示例 2：  
//  
//        输入：[2,7,9,3,1]  
//        输出：12  
//        解释：偷窃 1 号房屋 (金额 = 2), 偷窃 3 号房屋 (金额 = 9)，接着偷窃 5 号房屋 (金额 = 1)。  
//        偷窃到的最高金额 = 2 + 9 + 1 = 12 。  
//  
//  
//        提示：  
//  
//        1 <= nums.length <= 100  
//        0 <= nums[i] <= 400  
```
?
```java
public class Rob1 {  
	public int rob(int[] nums) {
	    if (nums.length == 0) {
	        return 0;
	    }
	    // 子问题：
	    // f(k) = 偷 [0..k) 房间中的最大金额
	
	    // f(0) = 0
	    // f(1) = nums[0]
	    // f(k) = max{ rob(k-1), nums[k-1] + rob(k-2) }
	
	    int N = nums.length;
	    int[] dp = new int[N+1];
	    dp[0] = 0;
	    dp[1] = nums[0];
	    for (int k = 2; k <= N; k++) {
	        dp[k] = Math.max(dp[k-1], nums[k-1] + dp[k-2]);
	    }
	    return dp[N];
	}

	public int rob(int[] nums) {
	    int prev = 0;
	    int curr = 0;
	
	    // 每次循环，计算“偷到当前房子为止的最大金额”
	    for (int i : nums) {
	        // 循环开始时，curr 表示 dp[k-1]，prev 表示 dp[k-2]
	        // dp[k] = max{ dp[k-1], dp[k-2] + i }
	        int temp = Math.max(curr, prev + i);
	        prev = curr;
	        curr = temp;
	        // 循环结束时，curr 表示 dp[k]，prev 表示 dp[k-1]
	    }
	
	    return curr;
	}
//    复杂度分析：  
//    时间复杂度 O(N) ： 遍历 nums 需要线性时间；  
//    空间复杂度 O(1) ： cur和 pre 使用常数大小的额外空间。  
}
```
<!--SR:!2023-08-16,6,85-->

```java
//213. 打家劫舍 II
//        你是一个专业的小偷，计划偷窃沿街的房屋，每间房内都藏有一定的现金。这个地方所有的房屋都 围成一圈 ，这意味着第一个房屋和最后一个房屋是紧挨着的。同时，相邻的房屋装有相互连通的防盗系统，如果两间相邻的房屋在同一晚上被小偷闯入，系统会自动报警 。  
//  
//        给定一个代表每个房屋存放金额的非负整数数组，计算你 在不触动警报装置的情况下 ，今晚能够偷窃到的最高金额。  
//  
//  
//  
//        示例 1：  
//  
//        输入：nums = [2,3,2]  
//        输出：3  
//        解释：你不能先偷窃 1 号房屋（金额 = 2），然后偷窃 3 号房屋（金额 = 2）, 因为他们是相邻的。  
//        示例 2：  
//  
//        输入：nums = [1,2,3,1]  
//        输出：4  
//        解释：你可以先偷窃 1 号房屋（金额 = 1），然后偷窃 3 号房屋（金额 = 3）。  
//        偷窃到的最高金额 = 1 + 3 = 4 。  
//        示例 3：  
//  
//        输入：nums = [1,2,3]  
//        输出：3  
//  
//  
//        提示：  
//  
//        1 <= nums.length <= 100  
//        0 <= nums[i] <= 1000  
```
?
```java
public class Rob {  
    public int rob(int[] nums) {  
        if(nums.length == 0) return 0;  
        if(nums.length == 1) return nums[0];  
        return Math.max(myRob(Arrays.copyOfRange(nums, 0, nums.length - 1)),  
                myRob(Arrays.copyOfRange(nums, 1, nums.length)));  
    }  
    private int myRob(int[] nums) {  
        int pre = 0, cur = 0, tmp;  
        for(int num : nums) {  
            tmp = cur;  
            cur = Math.max(pre + num, cur);  
            pre = tmp;  
        }  
        return cur;  
    }  
//    复杂度分析：  
//    时间复杂度 O(N) ： 两次遍历 nums 需要线性时间；  
//    空间复杂度 O(1) ： cur和 pre 使用常数大小的额外空间。  
}
```
<!--SR:!2023-10-03,9,130-->

```java
//467. 环绕字符串中唯一的子字符串  
//        把字符串 s 看作 "abcdefghijklmnopqrstuvwxyz" 的无限环绕字符串，所以 s 看起来是这样的：  
//  
//        "...zabcdefghijklmnopqrstuvwxyzabcdefghijklmnopqrstuvwxyzabcd...." 。  
//        现在给定另一个字符串 p 。返回 s 中 不同 的 p 的 非空子串 的数量 。  
//  
//  
//  
//        示例 1：  
//  
//        输入：p = "a"  
//        输出：1  
//        解释：字符串 s 中只有 p 的一个 "a" 子字符。  
//        示例 2：  
//  
//        输入：p = "cac"  
//        输出：2  
//        解释：字符串 s 中只有 p 的两个子串 ("a", "c") 。  
//        示例 3：  
//  
//        输入：p = "zab"  
//        输出：6  
//        解释：在字符串 s 中有 p 的六个子串 ("z", "a", "b", "za", "ab", "zab") 。  
//  
//  
//        提示：  
//  
//        1 <= p.length <= 105  
//        p 由小写英文字母组成  
```
?
```java
public class FindSubstringInWraproundString {  
    public int findSubstringInWraproundString(String p) {  
        int[] dp = new int[26];  
        int k = 0;  
        for (int i = 0; i < p.length(); ++i) {  
            if (i > 0 && (p.charAt(i) - p.charAt(i - 1) + 26) % 26 == 1) { // 字符之差为 1 或 -25                ++k;  
            } else {  
                k = 1;  
            }  
            dp[p.charAt(i) - 'a'] = Math.max(dp[p.charAt(i) - 'a'], k);  
        }  
        return Arrays.stream(dp).sum();  
    }  
//    复杂度分析  
//    时间复杂度：O(n)，其中 n 是字符串 p 的长度。  
//    空间复杂度：O(∣Σ∣)，其中 ∣Σ∣ 为字符集合的大小，本题中字符均为小写字母，故 ∣Σ∣=26。  
}
```
<!--SR:!2023-08-06,5,85-->


```java
//650. 只有两个键的键盘  
//        最初记事本上只有一个字符 'A' 。你每次可以对这个记事本进行两种操作：  
//  
//        Copy All（复制全部）：复制这个记事本中的所有字符（不允许仅复制部分字符）。  
//        Paste（粘贴）：粘贴 上一次 复制的字符。  
//        给你一个数字 n ，你需要使用最少的操作次数，在记事本上输出 恰好 n 个 'A' 。返回能够打印出 n 个 'A' 的最少操作次数。  
//  
//        示例 1：  
//        输入：3  
//        输出：3  
//        解释：  
//        最初, 只有一个字符 'A'。  
//        第 1 步, 使用 Copy All 操作。  
//        第 2 步, 使用 Paste 操作来获得 'AA'。  
//        第 3 步, 使用 Paste 操作来获得 'AAA'。  
//        示例 2：  
//        输入：n = 1  
//        输出：0  
//  
//        提示：  
//        1 <= n <= 1000  
```
?
```java
思路是：先反向思维，必然有一个操作是复制最多字符的，先获取这个操作之前的字符长度，就是n/2,以此反复，到最后必然是最小单位复制
public class MinSteps {  
    public static int minSteps(int n) {  
        int ans = 0;  
        for (int i = 2; i * i <= n; i++) {  
            while (n % i == 0) {  
                ans += i;  
                n /= i;  
            }  
        }  
        if (n != 1) ans += n;  
        return ans;  
    }  
//    时间复杂度：O(\sqrt{n})  
//    空间复杂度：O(1)  
}
```
<!--SR:!2023-10-13,23,85-->