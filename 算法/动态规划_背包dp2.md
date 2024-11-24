#review/second_101_算法 

```java
package other_code.动态规划.背包dp;  
//1049. 最后一块石头的重量 II//        有一堆石头，用整数数组 stones 表示。其中 stones[i] 表示第 i 块石头的重量。  
//  
//        每一回合，从中选出任意两块石头，然后将它们一起粉碎。假设石头的重量分别为 x 和 y，且 x <= y。那么粉碎的可能结果如下：  
//  
//        如果 x == y，那么两块石头都会被完全粉碎；  
//        如果 x != y，那么重量为 x 的石头将会完全粉碎，而重量为 y 的石头新重量为 y-x。  
//        最后，最多只会剩下一块 石头。返回此石头 最小的可能重量 。如果没有石头剩下，就返回 0。  
//  
//        示例 1：  
//        输入：stones = [2,7,4,1,8,1]  
//        输出：1  
//        解释：  
//        组合 2 和 4，得到 2，所以数组转化为 [2,7,1,8,1]，  
//        组合 7 和 8，得到 1，所以数组转化为 [2,1,1,1]，  
//        组合 2 和 1，得到 1，所以数组转化为 [1,1,1]，  
//        组合 1 和 1，得到 0，所以数组转化为 [1]，这就是最优值。  
//        示例 2：  
//        输入：stones = [31,26,33,21,40]  
//        输出：5  
//  
//        提示：  
//        1 <= stones.length <= 30  
//        1 <= stones[i] <= 100  

```
?
```java
public class LastStoneWeightII {  
    //普通版本  
    public int lastStoneWeightII(int[] stones) {  
        int n = stones.length;  
        int sum= 0;  
        for (int i : stones) sum += i;  
        int t = sum /2;  
        int[][] f = new int[n][t+1];  
        for (int i = 1;i<=n;i++){  
            int x = stones[i-1];  
            for (int j = 0 ;j<= t;j++){  
                f[i][j]=f[i-1][j];  
                if (j>=x) f[i][j] = Math.max(f[i][j],f[i-1][j-x]+x);  
            }  
        }  
        return Math.abs(sum-f[n][t]-f[n][t]);  
    }  
//滚动数组  
    public static int lastStoneWeightII1(int[] ss) {  
        int n = ss.length;  
        int sum = 0;  
        for (int i : ss) sum += i;  
        int t = sum / 2;  
        int[][] f = new int[2][t + 1];  
        for (int i = 1; i <= n; i++) {  
            int x = ss[i - 1];  
            int a = i & 1, b = (i - 1) & 1;  
            for (int j = 0; j <= t; j++) {  
                f[a][j] = f[b][j];  
                if (j >= x) f[a][j] = Math.max(f[a][j], f[b][j - x] + x);  
            }  
        }  
        return Math.abs(sum - f[n&1][t] - f[n&1][t]);  
    }  
    //一维数组空间优化 
     //去掉第一个维度之后，若还是采用正序遍历，在计算dp[i]的时候，dp[j-stones[i]]的值已经被覆盖，这意味这dp[j-stones[i]]实际对应的是dp[i+1][j-stones[i]],即我们计算的是一个错误的转移方程：
     //dp[i+1][j]=dp[i][j]Vdp[i+1][j-stones[i]]
    public int lastStoneWeightII2(int[] ss) {  
        int n = ss.length;  
        int sum = 0;  
        for (int i : ss) sum += i;  
        int t = sum / 2;  
        int[] f = new int[t + 1];  
        for (int i = 1; i <= n; i++) {  
            int x = ss[i - 1];  
            for (int j = t; j >= x; j--) {  
                f[j] = Math.max(f[j], f[j - x] + x);  
            }  
        }  
        return Math.abs(sum - f[t] - f[t]);  
    }  
}
```
<!--SR:!2022-10-02,1,210-->

```java
#### [638. 大礼包](https://leetcode.cn/problems/shopping-offers/)

难度中等346

在 LeetCode 商店中， 有 `n` 件在售的物品。每件物品都有对应的价格。然而，也有一些大礼包，每个大礼包以优惠的价格捆绑销售一组物品。

给你一个整数数组 `price` 表示物品价格，其中 `price[i]` 是第 `i` 件物品的价格。另有一个整数数组 `needs` 表示购物清单，其中 `needs[i]` 是需要购买第 `i` 件物品的数量。

还有一个数组 `special` 表示大礼包，`special[i]` 的长度为 `n + 1` ，其中 `special[i][j]` 表示第 `i` 个大礼包中内含第 `j` 件物品的数量，且 `special[i][n]` （也就是数组中的最后一个整数）为第 `i` 个大礼包的价格。

返回 **确切** 满足购物清单所需花费的最低价格，你可以充分利用大礼包的优惠活动。你不能购买超出购物清单指定数量的物品，即使那样会降低整体价格。任意大礼包可无限次购买。

**示例 1：**

**输入：**price = [2,5], special = [[3,0,5],[1,2,10]], needs = [3,2]
**输出：**14
**解释：**有 A 和 B 两种物品，价格分别为 ¥2 和 ¥5 。 
大礼包 1 ，你可以以 ¥5 的价格购买 3A 和 0B 。 
大礼包 2 ，你可以以 ¥10 的价格购买 1A 和 2B 。 
需要购买 3 个 A 和 2 个 B ， 所以付 ¥10 购买 1A 和 2B（大礼包 2），以及 ¥4 购买 2A 。

**示例 2：**

**输入：**price = [2,3,4], special = [[1,1,0,4],[2,2,1,9]], needs = [1,2,1]
**输出：**11
**解释：**A ，B ，C 的价格分别为 ¥2 ，¥3 ，¥4 。
可以用 ¥4 购买 1A 和 1B ，也可以用 ¥9 购买 2A ，2B 和 1C 。 
需要买 1A ，2B 和 1C ，所以付 ¥4 买 1A 和 1B（大礼包 1），以及 ¥3 购买 1B ， ¥4 购买 1C 。 
不可以购买超出待购清单的物品，尽管购买大礼包 2 更加便宜。

**提示：**

-   `n == price.length`
-   `n == needs.length`
-   `1 <= n <= 6`
-   `0 <= price[i] <= 10`
-   `0 <= needs[i] <= 10`
-   `1 <= special.length <= 100`
-   `special[i].length == n + 1`
-   `0 <= special[i][j] <= 50`
```
?
```bash
class Solution {
public:
    // 不同的needs所需的价格
    map<vector<int>, int> _cache;

    int shoppingOffers(vector<int>& price, vector<vector<int>>& special, vector<int>& needs) {
        return dfs(needs, price, special);
    }

    int dfs(vector<int> needs, vector<int>& price, vector<vector<int>>& special) {
        // 如果子问题已经计算过 直接返回
        if (_cache[needs]) { return _cache[needs]; }
        int ans = 0;
        // 最弱的方式；不购买大礼包
        for (int i = 0; i < needs.size(); i++) {
            ans += needs[i] * price[i];
        }

        // 遍历每个礼包，购买它，看看是不是能获得更便宜的价格
        for (int i = 0; i < special.size(); i++) {
            vector<int> next = needs;
            bool valid = true;
            // 因为购买的数量需要正好是needs 所以大礼包的某个商品不能超过needs的商品数量
            for (int item = 0; item < price.size(); item++) {
                if (special[i][item] > needs[item]) {
                    valid = false;
                    break;
                }
            }

            // 当前大礼包不符合要求 跳过
            if (!valid) continue;
            // 当前大礼包符合要求，用next数组记录买过大礼包之后还需要买多少商品
            for (int item = 0; item < price.size(); item++) {
                next[item] -= special[i][item];
            }

            // 在整个遍历过程中，我们要找价格最小的一种方式
            ans = min(ans, dfs(next, price, special) + special[i].back());
        }

        // 更新cache
        _cache[needs] = ans;
        return ans;
    }

};

时间复杂度：令物品数量为 nn，原礼包数量为 mm。每个物品最多需要 k =max(needs[i])=10 个，共有 k^n个状态需要转移，转移时需要考虑「单买」和「礼包」决策，复杂度分别为 O(n) 和 O(m∗n)。整体复杂度为 O(k^n * (m * n))
空间复杂度：O(k^n * (m * n))
```
<!--SR:!2022-09-27,1,230-->


```java
#### [474. 一和零](https://leetcode.cn/problems/ones-and-zeroes/)

难度中等802

给你一个二进制字符串数组 `strs` 和两个整数 `m` 和 `n` 。

请你找出并返回 `strs` 的最大子集的长度，该子集中 **最多** 有 `m` 个 `0` 和 `n` 个 `1` 。

如果 `x` 的所有元素也是 `y` 的元素，集合 `x` 是集合 `y` 的 **子集** 。

**示例 1：**

**输入：**strs = ["10", "0001", "111001", "1", "0"], m = 5, n = 3
**输出：**4
**解释：**最多有 5 个 0 和 3 个 1 的最大子集是 {"10","0001","1","0"} ，因此答案是 4 。
其他满足题意但较小的子集包括 {"0001","1"} 和 {"10","1","0"} 。{"111001"} 不满足题意，因为它含 4 个 1 ，大于 n 的值 3 。

**示例 2：**

**输入：**strs = ["10", "0", "1"], m = 1, n = 1
**输出：**2
**解释：**最大的子集是 {"0", "1"} ，所以答案是 2 。

**提示：**

-   `1 <= strs.length <= 600`
-   `1 <= strs[i].length <= 100`
-   `strs[i]` 仅由 `'0'` 和 `'1'` 组成
-   `1 <= m, n <= 100`
```
?
```java
开始动规五部曲：

1.  确定dp数组（dp table）以及下标的含义

**dp[i][j]：最多有i个0和j个1的strs的最大子集的大小为dp[i][j]**。

2.  确定递推公式

dp[i][j] 可以由前一个strs里的字符串推导出来，strs里的字符串有zeroNum个0，oneNum个1。

dp[i][j] 就可以是 dp[i - zeroNum][j - oneNum] + 1。

然后我们在遍历的过程中，取dp[i][j]的最大值。

所以递推公式：dp[i][j] = max(dp[i][j], dp[i - zeroNum][j - oneNum] + 1);

此时大家可以回想一下01背包的递推公式：dp[j] = max(dp[j], dp[j - weight[i]] + value[i]);

对比一下就会发现，字符串的zeroNum和oneNum相当于物品的重量（weight[i]），字符串本身的个数相当于物品的价值（value[i]）。
class Solution {
    public int findMaxForm(String[] strs, int m, int n) {
        //dp[i][j]表示i个0和j个1时的最大子集
        int[][] dp = new int[m + 1][n + 1];
        int oneNum, zeroNum;
        for (String str : strs) {
            oneNum = 0;
            zeroNum = 0;
            for (char ch : str.toCharArray()) {
                if (ch == '0') {
                    zeroNum++;
                } else {
                    oneNum++;
                }
            }
            //倒序遍历
            for (int i = m; i >= zeroNum; i--) {
                for (int j = n; j >= oneNum; j--) {
                    dp[i][j] = Math.max(dp[i][j], dp[i - zeroNum][j - oneNum] + 1);
                }
            }
        }
        return dp[m][n];
    }
}
复杂度分析

时间复杂度：O(lmn+L)，其中 l 是数组 strs 的长度，m 和 n 分别是 0 和 1 的容量，L 是数组 strs 中的所有字符串的长度之和。
动态规划需要计算的状态总数是 O(lmn)，每个状态的值需要 O(1) 的时间计算。
对于数组 strs 中的每个字符串，都要遍历字符串得到其中的 00 和 11 的数量，因此需要 O(L) 的时间遍历所有的字符串。
总时间复杂度是 O(lmn+L)。

空间复杂度：O(mn)，其中 mm 和 nn 分别是 00 和 11 的容量。使用空间优化的实现，需要创建 m+1 行 n+1 列的二维数组 dp。

```
<!--SR:!2022-10-10,2,210-->

```java
#### [518. 零钱兑换 II](https://leetcode.cn/problems/coin-change-2/)

难度中等922收藏分享切换为英文接收动态反馈

给你一个整数数组 `coins` 表示不同面额的硬币，另给一个整数 `amount` 表示总金额。

请你计算并返回可以凑成总金额的硬币组合数。如果任何硬币组合都无法凑出总金额，返回 `0` 。

假设每一种面额的硬币有无限个。 

题目数据保证结果符合 32 位带符号整数。

**示例 1：**

**输入：**amount = 5, coins = [1, 2, 5]
**输出：**4
**解释：**有四种方式可以凑成总金额：
5=5
5=2+2+1
5=2+1+1+1
5=1+1+1+1+1

**示例 2：**

**输入：**amount = 3, coins = [2]
**输出：**0
**解释：**只用面额 2 的硬币不能凑成总金额 3 。

**示例 3：**

**输入：**amount = 10, coins = [10] 
**输出：**1

**提示：**

-   `1 <= coins.length <= 300`
-   `1 <= coins[i] <= 5000`
-   `coins` 中的所有值 **互不相同**
-   `0 <= amount <= 5000`
```
?
```java
由此可以得到动态规划的做法：
初始化dp[0]=1；
遍历 coins，对于其中的每个元素 coin，进行如下操作：
遍历 i 从 coin 到 amount，将 dp[i−coin] 的值加到 dp[i]。
最终得到 dp[amount] 的值即为答案。

class Solution {
    public int change(int amount, int[] coins) {
        int[] dp = new int[amount + 1];
        dp[0] = 1;
        for (int coin : coins) {
            for (int i = coin; i <= amount; i++) {
                dp[i] += dp[i - coin];
            }
        }
        return dp[amount];
    }
}

复杂度分析

时间复杂度：O(amount×n)，其中 amount 是总金额，n 是数组 coins 的长度。需要使用数组 coins 中的每个元素遍历并更新数组dp 中的每个元素的值。

空间复杂度:O(amount)，其中amount 是总金额。需要创建长度为 amount+1 的数组 dp。
```
<!--SR:!2022-10-02,1,210-->