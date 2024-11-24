#review/third_101_算法 

```java
#### [1022. 从根到叶的二进制数之和](https://leetcode.cn/problems/sum-of-root-to-leaf-binary-numbers/)

给出一棵二叉树，其上每个结点的值都是 `0` 或 `1` 。每一条从根到叶的路径都代表一个从最高有效位开始的二进制数。

-   例如，如果路径为 `0 -> 1 -> 1 -> 0 -> 1`，那么它表示二进制数 `01101`，也就是 `13` 。

对树上的每一片叶子，我们都要找出从根到该叶子的路径所表示的数字。

返回这些数字之和。题目数据保证答案是一个 **32 位** 整数
输入：root = [1,0,1,0,1,0,1]
输出：22
解释：(100) + (101) + (110) + (111) = 4 + 5 + 6 + 7 = 22
示例 2：

输入：root = [0]
输出：0

```
?
```java
递归
class Solution {
    public int sumRootToLeaf(TreeNode root) {
        return dfs(root, 0);
    }
    int dfs(TreeNode root, int cur) {
        int ans = 0, ncur = (cur << 1) + root.val;
        if (root.left != null) ans += dfs(root.left, ncur);
        if (root.right != null) ans += dfs(root.right, ncur);
        return root.left == null && root.right == null ? ncur : ans;
    }
}

时间复杂度：O(n)
空间复杂度：忽略递归带来的额外空间开销，复杂度为 O(1)

迭代
class Solution {
    public int sumRootToLeaf(TreeNode root) {
        int ans = 0;
        Deque<TreeNode> d = new ArrayDeque<>();
        d.addLast(root);
        while (!d.isEmpty()) {
            TreeNode poll = d.pollFirst();
            if (poll.left != null) {
                poll.left.val = (poll.val << 1) + poll.left.val;
                d.addLast(poll.left);
            }
            if (poll.right != null) {
                poll.right.val = (poll.val << 1) + poll.right.val;
                d.addLast(poll.right);
            }
            if (poll.left == null && poll.right == null) ans += poll.val;
        }
        return ans;
    }
}

-   时间复杂度：O(n)
-   空间复杂度：O(n)

```
<!--SR:!2024-01-22,12,230-->

```java
#### [993. 二叉树的堂兄弟节点](https://leetcode.cn/problems/cousins-in-binary-tree/)

难度简单278

在二叉树中，根节点位于深度 `0` 处，每个深度为 `k` 的节点的子节点位于深度 `k+1` 处。

如果二叉树的两个节点深度相同，但 **父节点不同** ，则它们是一对_堂兄弟节点_。

我们给出了具有唯一值的二叉树的根节点 `root` ，以及树中两个不同节点的值 `x` 和 `y` 。

只有与值 `x` 和 `y` 对应的节点是堂兄弟节点时，才返回 `true` 。否则，返回 `false`。
**提示：**

-   二叉树的节点数介于 `2` 到 `100` 之间。
-   每个节点的值都是唯一的、范围为 `1` 到 `100` 的整数。
```
?
```java
DFS
class Solution {
    public boolean isCousins(TreeNode root, int x, int y) {
        int[] xi = dfs(root, null, 0, x);
        int[] yi = dfs(root, null, 0, y);
        return xi[1] == yi[1] && xi[0] != yi[0];
    }
    int[] dfs(TreeNode root, TreeNode fa, int depth, int t) {
        if (root == null) return new int[]{-1, -1}; // 使用 -1 代表为搜索不到 t
        if (root.val == t) {
            return new int[]{fa != null ? fa.val : 1, depth}; // 使用 1 代表搜索值 t 为 root
        }
        int[] l = dfs(root.left, root, depth + 1, t);
        if (l[0] != -1) return l;
        return dfs(root.right, root, depth + 1, t);
    }
}
-   时间复杂度：O(n)
-   空间复杂度：忽略递归开销为 O(1)，否则为 O(n)

BFS
class Solution {
    public boolean isCousins(TreeNode root, int x, int y) {
        int[] xi = bfs(root, x);
        int[] yi = bfs(root, y);
        return xi[1] == yi[1] && xi[0] != yi[0];
    }
    int[] bfs(TreeNode root, int t) {
        Deque<Object[]> d = new ArrayDeque<>(); // 存储值为 [cur, fa, depth]
        d.addLast(new Object[]{root, null, 0});
        while (!d.isEmpty()) {
            int size = d.size();
            while (size-- > 0) {
                Object[] poll = d.pollFirst();
                TreeNode cur = (TreeNode)poll[0], fa = (TreeNode)poll[1];
                int depth = (Integer)poll[2];

                if (cur.val == t) return new int[]{fa != null ? fa.val : 0, depth};
                if (cur.left != null) d.addLast(new Object[]{cur.left, cur, depth + 1});
                if (cur.right != null) d.addLast(new Object[]{cur.right, cur, depth + 1});
            }
        }
        return new int[]{-1, -1};
    }
}

-   时间复杂度：O(n)
-   空间复杂度：O(n)

```
<!--SR:!2024-01-22,12,230-->

```java
#### [965. 单值二叉树](https://leetcode.cn/problems/univalued-binary-tree/)

难度简单171

如果二叉树每个节点都具有相同的值，那么该二叉树就是_单值_二叉树。

只有给定的树是单值二叉树时，才返回 `true`；否则返回 `false`。

```
?
```java
//递归
class Solution {
    int val = -1;
    public boolean isUnivalTree(TreeNode root) {
        if (val == -1) val = root.val;
        if (root == null) return true;
        if (root.val != val) return false;
        return isUnivalTree(root.left) && isUnivalTree(root.right);
    }
}
//迭代
class Solution {
    public boolean isUnivalTree(TreeNode root) {
        int val = root.val;
        Deque<TreeNode> d = new ArrayDeque<>();
        d.addLast(root);
        while (!d.isEmpty()) {
            TreeNode poll = d.pollFirst();
            if (poll.val != val) return false;
            if (poll.left != null) d.addLast(poll.left);
            if (poll.right != null) d.addLast(poll.right);
        }
        return true;
    }
}
-   时间复杂度：O(n)
-   空间复杂度：O(n)
```
<!--SR:!2024-01-27,1,230-->

```java
#### [938. 二叉搜索树的范围和](https://leetcode.cn/problems/range-sum-of-bst/)

给定二叉搜索树的根结点 `root`，返回值位于范围 _`[low, high]`_ 之间的所有结点的值的和。
```
?
```java
#### 方法一：深度优先搜索
class Solution {
    public int rangeSumBST(TreeNode root, int low, int high) {
        if (root == null) {
            return 0;
        }
        if (root.val > high) {
            return rangeSumBST(root.left, low, high);
        }
        if (root.val < low) {
            return rangeSumBST(root.right, low, high);
        }
        return root.val + rangeSumBST(root.left, low, high) + rangeSumBST(root.right, low, high);
    }
}

**复杂度分析**
-   时间复杂度：O(n)，其中 nn 是二叉搜索树的节点数。
-   空间复杂度：O(n)。空间复杂度主要取决于栈空间的开销。

#### 方法二：广度优先搜索
class Solution {
    public int rangeSumBST(TreeNode root, int low, int high) {
        int sum = 0;
        Queue<TreeNode> q = new LinkedList<TreeNode>();
        q.offer(root);
        while (!q.isEmpty()) {
            TreeNode node = q.poll();
            if (node == null) {
                continue;
            }
            if (node.val > high) {
                q.offer(node.left);
            } else if (node.val < low) {
                q.offer(node.right);
            } else {
                sum += node.val;
                q.offer(node.left);
                q.offer(node.right);
            }
        }
        return sum;
    }
}
**复杂度分析**
-   时间复杂度：O(n)，其中 n 是二叉搜索树的节点数。
-   空间复杂度：O(n)。空间复杂度主要取决于队列的空间。
```
<!--SR:!2024-02-27,1,230-->

```java
#### [821. 字符的最短距离](https://leetcode.cn/problems/shortest-distance-to-a-character/)

给你一个字符串 s 和一个字符 c ，且 c 是 s 中出现过的字符。

返回一个整数数组 answer ，其中 answer.length == s.length 且 answer[i] 是 s中从下标 i 到离它最近的字符 c的距离 。

两个下标 i 和 j 之间的 距离 为 abs(i - j) ，其中 abs 是绝对值函数。
示例 1：

输入：s = "loveleetcode", c = "e"
输出：[3,2,1,0,1,0,0,1,2,2,1,0]
解释：字符 'e' 出现在下标 3、5、6 和 11 处（下标从 0 开始计数）。
距下标 0 最近的 'e' 出现在下标 3 ，所以距离为 abs(0 - 3) = 3 。
距下标 1 最近的 'e' 出现在下标 3 ，所以距离为 abs(1 - 3) = 2 。
对于下标 4 ，出现在下标 3 和下标 5 处的 'e' 都离它最近，但距离是一样的 abs(4 - 3) == abs(4 - 5) = 1 。
距下标 8 最近的 'e' 出现在下标 6 ，所以距离为 abs(8 - 6) = 2 。
```
?
```java
class Solution {
    public int[] shortestToChar(String s, char c) {
        int n = s.length();
        int[] ans = new int[n];

        for (int i = 0, idx = -n; i < n; ++i) {
            if (s.charAt(i) == c) {
                idx = i;
            }
            ans[i] = i - idx;
        }

        for (int i = n - 1, idx = 2 * n; i >= 0; --i) {
            if (s.charAt(i) == c) {
                idx = i;
            }
            ans[i] = Math.min(ans[i], idx - i);
        }
        return ans;
    }
}

复杂度分析
-   时间复杂度：O(n)，其中 n 是字符串 s 的长度。
-   空间复杂度：O(1)。返回值不计入空间复杂度。
```
<!--SR:!2024-02-26,7,230-->


```java
#### [590. N 叉树的后序遍历](https://leetcode.cn/problems/n-ary-tree-postorder-traversal/)

难度简单266

给定一个 n 叉树的根节点 `root` ，返回 _其节点值的 **后序遍历**_ 。

n 叉树 在输入中按层序遍历进行序列化表示，每组子节点由空值 `null` 分隔（请参见示例）。
```
?
```java
#### 方法一：递归
class Solution {
    public List<Integer> postorder(Node root) {
        List<Integer> res = new ArrayList<>();
        helper(root, res);
        return res;
    }

    public void helper(Node root, List<Integer> res) {
        if (root == null) {
            return;
        }
        for (Node ch : root.children) {
            helper(ch, res);
        }
        res.add(root.val);
    }
}

#### 方法四：利用前序遍历反转
其中 children(v_k)表示以 v_k
  为根节点的子树的遍历结果（不包括 v_k)。仔细观察可以知道，将前序遍历中子树的访问顺序改为从右向左可以得到如下访问顺序：
[u,v3,children(v3),v2,children(v2),v1,children(v1)]
将上述的结果进行反转，得到:
[children(v1),v1,children(v2),v2,children(v3),v3,u]

刚好与后续遍历的结果相同。此时我们可以利用前序遍历，只不过前序遍历中对子节点的遍历顺序是从左向右，而这里是从右向左。因此我们可以使用和 NN 叉树的前序遍历相同的方法，使用一个栈来得到后序遍历。

我们首先把根节点入栈。当每次我们从栈顶取出一个节点 uu 时，就把 uu 的所有子节点顺序推入栈中。例如 uu 的子节点从左到右为 v_1, v_2, v_3
 ，那么推入栈的顺序应当为 v_1, v_2, v_3
 ，这样就保证了出栈顺序是从右向左，下一个遍历到的节点（即 uu 的右侧第一个子节点 v_3）出现在栈顶的位置。在遍历结束之后，我们把遍历结果进行反转，就可以得到后序遍历。
class Solution {
    public List<Integer> postorder(Node root) {
        List<Integer> res = new ArrayList<>();
        if (root == null) {
            return res;
        }

        Deque<Node> stack = new ArrayDeque<Node>();
        stack.push(root);
        while (!stack.isEmpty()) {
            Node node = stack.pop();
            res.add(node.val);
            for (Node item : node.children) {
                stack.push(item);
            }
        }
        Collections.reverse(res);
        return res;
    }
}
```
<!--SR:!2024-01-22,16,230-->

```java
#### [589. N 叉树的前序遍历](https://leetcode.cn/problems/n-ary-tree-preorder-traversal/)

给定一个 n 叉树的根节点  `root` ，返回 _其节点值的 **前序遍历**_ 。

n 叉树 在输入中按层序遍历进行序列化表示，每组子节点由空值 `null` 分隔（请参见示例）。
```
?
```java
/*
// Definition for a Node.
class Node {
    public int val;
    public List<Node> children;
    public Node() {}
    public Node(int _val) {
        val = _val;
    }

    public Node(int _val, List<Node> _children) {
        val = _val;
        children = _children;
    }

};

*/

#### 方法一：递归
class Solution {
    public List<Integer> preorder(Node root) {
        List<Integer> res = new ArrayList<>();
        helper(root, res);
        return res;
    }

    public void helper(Node root, List<Integer> res) {
        if (root == null) {
            return;
        }
        res.add(root.val);
        for (Node ch : root.children) {
            helper(ch, res);
        }
    }
}

复杂度分析

时间复杂度：O(m)，其中 m 为 N 叉树的节点。每个节点恰好被遍历一次。

空间复杂度：O(m)，递归过程中需要调用栈的开销，平均情况下为 O(logm)，最坏情况下树的深度为 m−1，此时需要的空间复杂度为 O(m)。

#### 方法三：迭代优化
class Solution {
    public List<Integer> preorder(Node root) {
        List<Integer> res = new ArrayList<>();
        if (root == null) {
            return res;
        }

        Deque<Node> stack = new ArrayDeque<Node>();
        stack.push(root);
        while (!stack.isEmpty()) {
            Node node = stack.pop();
            res.add(node.val);
            for (int i = node.children.size() - 1; i >= 0; --i) {
                stack.push(node.children.get(i));
            }
        }
        return res;
    }
}

复杂度分析

时间复杂度：O(m)，其中 m 为 N 叉树的节点。每个节点恰好被访问一次。

空间复杂度：O(m)，其中 m 为 N 叉树的节点。如果 N 叉树的深度为 1 则此时栈的空间为 O(m−1)，如果 N 叉树的深度为 m−1 则此时栈的空间为 O(1)，平均情况下栈的空间为 O(logm)，因此空间复杂度为 O(m)。

```
<!--SR:!2024-01-22,7,230-->

```java
#### [559. N 叉树的最大深度](https://leetcode.cn/problems/maximum-depth-of-n-ary-tree/)

给定一个 N 叉树，找到其最大深度。

最大深度是指从根节点到最远叶子节点的最长路径上的节点总数。

N 叉树输入按层序遍历序列化表示，每组子节点由空值分隔（请参见示例）。
```
?
```java
方法一：深度优先搜索
如果根节点有 N 个子节点，则这 N 个子节点对应 N 个子树。记 N 个子树的最大深度中的最大值为 maxChildDepth，则该 N 叉树的最大深度为 maxChildDepth+1。

每个子树的最大深度又可以以同样的方式进行计算。因此我们可以用「深度优先搜索」的方法计算 N 叉树的最大深度。具体而言，在计算当前 N 叉树的最大深度时，可以先递归计算出其每个子树的最大深度，然后在 O(1) 的时间内计算出当前 N 叉树的最大深度。递归在访问到空节点时退出。

class Solution {
    public int maxDepth(Node root) {
        if (root == null) {
            return 0;
        }
        int maxChildDepth = 0;
        List<Node> children = root.children;
        for (Node child : children) {
            int childDepth = maxDepth(child);
            maxChildDepth = Math.max(maxChildDepth, childDepth);
        }
        return maxChildDepth + 1;
    }
}
复杂度分析

时间复杂度：O(n)，其中 n 为 N 叉树节点的个数。每个节点在递归中只被遍历一次。

空间复杂度：O(height)，其中 height 表示 N 叉树的高度。递归函数需要栈空间，而栈空间取决于递归的深度，因此空间复杂度等价于 N 叉树的高度。

方法二：广度优先搜索
我们也可以用「广度优先搜索」的方法来解决这道题目，但我们需要对其进行一些修改，此时我们广度优先搜索的队列里存放的是「当前层的所有节点」。每次拓展下一层的时候，不同于广度优先搜索的每次只从队列里拿出一个节点，我们需要将队列里的所有节点都拿出来进行拓展，这样能保证每次拓展完的时候队列里存放的是当前层的所有节点，即我们是一层一层地进行拓展。最后我们用一个变量 ans 来维护拓展的次数，该 N 叉树的最大深度即为 ans。

class Solution {
    public int maxDepth(Node root) {
        if (root == null) {
            return 0;
        }
        Queue<Node> queue = new LinkedList<Node>();
        queue.offer(root);
        int ans = 0;
        while (!queue.isEmpty()) {
            int size = queue.size();
            while (size > 0) {
                Node node = queue.poll();
                List<Node> children = node.children;
                for (Node child : children) {
                    queue.offer(child);
                }
                size--;
            }
            ans++;
        }
        return ans;
    }
}

复杂度分析
时间复杂度：O(n)，其中 n 为 N 叉树的节点个数。与方法一同样的分析，每个节点只会被访问一次。
空间复杂度：此方法空间的消耗取决于队列存储的元素数量，其在最坏情况下会达到 O(n)。

```
<!--SR:!2024-01-22,15,230-->

```java
#### [剑指 Offer 32 - I. 从上到下打印二叉树](https://leetcode.cn/problems/cong-shang-dao-xia-da-yin-er-cha-shu-lcof/)

从上到下打印出二叉树的每个节点，同一层的节点按照从左到右的顺序打印。
```
?
```java
/**
 * Definition for a binary tree node.
 * public class TreeNode {
 *     int val;
 *     TreeNode left;
 *     TreeNode right;
 *     TreeNode(int x) { val = x; }
 * }
 */
## 迭代 - BFS

使用「迭代」进行求解是容易的，只需使用常规的 `BFS` 方法进行层序遍历即可。
class Solution {
    public int[] levelOrder(TreeNode root) {
        List<Integer> list = new ArrayList<>();
        Deque<TreeNode> d = new ArrayDeque<>();
        if (root != null) d.addLast(root);
        while (!d.isEmpty()) {
            TreeNode t = d.pollFirst();
            list.add(t.val);
            if (t.left != null) d.addLast(t.left);
            if (t.right != null) d.addLast(t.right);
        }
        int n = list.size();
        int[] ans = new int[n];
        for (int i = 0; i < n; i++) ans[i] = list.get(i);
        return ans;
    }
}

-   时间复杂度：O(n)
-   空间复杂度：O(n)
递归 - DFS
使用「递归」来进行「层序遍历」虽然不太符合直观印象，但也是可以的。

此时我们需要借助「哈希表」来存储起来每一层的节点情况。

首先我们按照「中序遍历」的方式进行 DFS，同时在 DFS 过程中传递节点所在的深度（root 节点默认在深度最小的第 00 层），每次处理当前节点时，通过哈希表获取所在层的数组，并将当前节点值追加到数组尾部，同时维护一个最大深度 max，在 DFS 完成后，再使用深度范围 [0, max][0,max] 从哈希表中进行构造答案。

class Solution {
    Map<Integer, List<Integer>> map = new HashMap<>();
    int max = -1, cnt = 0;
    public int[] levelOrder(TreeNode root) {
        dfs(root, 0);
        int[] ans = new int[cnt];
        for (int i = 0, idx = 0; i <= max; i++) {
            for (int x : map.get(i)) ans[idx++] = x;
        }
        return ans;
    }
    void dfs(TreeNode root, int depth) {
        if (root == null) return ;
        max = Math.max(max, depth);
        cnt++;
        dfs(root.left, depth + 1);
        List<Integer> list = map.getOrDefault(depth, new ArrayList<Integer>());
        list.add(root.val);
        map.put(depth, list);
        dfs(root.right, depth + 1);
    }
}

-   时间复杂度：O(n)
-   空间复杂度：O(n)

```
<!--SR:!2024-01-22,15,230-->

```java
#### [515. 在每个树行中找最大值](https://leetcode.cn/problems/find-largest-value-in-each-tree-row/)

给定一棵二叉树的根节点 `root` ，请找出该二叉树中每一层的最大值。
```
?
```java
//BFS
class Solution {
    public List<Integer> largestValues(TreeNode root) {
        List<Integer> ans = new ArrayList<>();
        if (root == null) return ans;
        Deque<TreeNode> d = new ArrayDeque<>();
        d.addLast(root);
        while (!d.isEmpty()) {
            int sz = d.size(), max = d.peek().val;
            while (sz-- > 0) {
                TreeNode node = d.pollFirst();
                max = Math.max(max, node.val);
                if (node.left != null) d.addLast(node.left);
                if (node.right != null) d.addLast(node.right);
            }
            ans.add(max);
        }
        return ans;
    }
}
//DFS
class Solution {
    int max = 0;
    Map<Integer, Integer> map = new HashMap<>();
    public List<Integer> largestValues(TreeNode root) {
        List<Integer> ans = new ArrayList<>();
        dfs(root, 1);
        for (int i = 1; i <= max; i++) ans.add(map.get(i));
        return ans;
    }
    void dfs(TreeNode node, int depth) {
        if (node == null) return ;
        max = Math.max(max, depth);
        map.put(depth, Math.max(map.getOrDefault(depth, Integer.MIN_VALUE), node.val));
        dfs(node.left, depth + 1);
        dfs(node.right, depth + 1);
    }
}

```
<!--SR:!2024-01-28,7,230-->

```java
#### [513. 找树左下角的值](https://leetcode.cn/problems/find-bottom-left-tree-value/)

给定一个二叉树的 根节点 `root`，请找出该二叉树的 最底层 最左边 节点的值。

假设二叉树中至少有一个节点。
```
?
```java
方法一：深度优先搜索
使用height 记录遍历到的节点的高度，curVal 记录高度在 curHeight 的最左节点的值。在深度优先搜索时，我们先搜索当前节点的左子节点，再搜索当前节点的右子节点，然后判断当前节点的高度 height 是否大于 curHeight，如果是，那么将 curVal 设置为当前结点的值，curHeight 设置为 height。

因为我们先遍历左子树，然后再遍历右子树，所以对同一高度的所有节点，最左节点肯定是最先被遍历到的。
class Solution {
    int curVal = 0;
    int curHeight = 0;

    public int findBottomLeftValue(TreeNode root) {
        int curHeight = 0;
        dfs(root, 0);
        return curVal;
    }

    public void dfs(TreeNode root, int height) {
        if (root == null) {
            return;
        }
        height++;
        dfs(root.left, height);
        dfs(root.right, height);
        if (height > curHeight) {
            curHeight = height;
            curVal = root.val;
        }
    }
}
复杂度分析
时间复杂度：O(n)，其中 n 是二叉树的节点数目。需要遍历 n 个节点。
空间复杂度：O(n)。递归栈需要占用 O(n) 的空间。

方法二：广度优先搜索
使用广度优先搜索遍历每一层的节点。在遍历一个节点时，需要先把它的非空右子节点放入队列，然后再把它的非空左子节点放入队列，这样才能保证从右到左遍历每一层的节点。广度优先搜索所遍历的最后一个节点的值就是最底层最左边节点的值。
class Solution {
    public int findBottomLeftValue(TreeNode root) {
        int ret = 0;
        Queue<TreeNode> queue = new ArrayDeque<TreeNode>();
        queue.offer(root);
        while (!queue.isEmpty()) {
            TreeNode p = queue.poll();
            if (p.right != null) {
                queue.offer(p.right);
            }
            if (p.left != null) {
                queue.offer(p.left);
            }
            ret = p.val;
        }
        return ret;
    }
}
复杂度分析
时间复杂度：O(n)，其中 n 是二叉树的节点数目。
空间复杂度：O(n)。如果二叉树是满完全二叉树，那么队列 q 最多保存 n/2 个节点。

```
<!--SR:!2024-02-25,17,230-->

```java
#### [397. 整数替换](https://leetcode.cn/problems/integer-replacement/)

给定一个正整数 `n` ，你可以做如下操作：
1.  如果 `n` 是偶数，则用 `n / 2`替换 `n` 。
2.  如果 `n` 是奇数，则可以用 `n + 1`或`n - 1`替换 `n` 。

返回 `n` 变为 `1` 所需的 _最小替换次数_ 。

**示例 1：**

**输入：**n = 8
**输出：**3
**解释：**8 -> 4 -> 2 -> 1

**示例 2：**

**输入：**n = 7
**输出：**4
**解释：**7 -> 8 -> 4 -> 2 -> 1
或 7 -> 6 -> 3 -> 2 -> 1

**示例 3：**

**输入：**n = 4
**输出：**2

**提示：**

-   `1 <= n <= 231 - 1`
```
?
```java
//记忆化搜索
class Solution {
    Map<Integer, Integer> memo = new HashMap<Integer, Integer>();

    public int integerReplacement(int n) {
        if (n == 1) {
            return 0;
        }
        if (!memo.containsKey(n)) {
            if (n % 2 == 0) {
                memo.put(n, 1 + integerReplacement(n / 2));
            } else {
                memo.put(n, 2 + Math.min(integerReplacement(n / 2), integerReplacement(n / 2 + 1)));
            }
        }
        return memo.get(n);
    }
}
复杂度分析

时间复杂度：O(logn)。

空间复杂度：O(logn)，记忆化搜索需要的空间与栈空间相同，同样为 O(logn)。

```
<!--SR:!2024-10-12,4,210-->


```java
#### [78. 子集](https://leetcode.cn/problems/subsets/)

给你一个整数数组 `nums` ，数组中的元素 **互不相同** 。返回该数组所有可能的子集（幂集）。

解集 **不能** 包含重复的子集。你可以按 **任意顺序** 返回解集。

**示例 1：**

**输入：**nums = [1,2,3]
**输出：**[[],[1],[2],[1,2],[3],[1,3],[2,3],[1,2,3]]

**示例 2：**

**输入：**nums = [0]
**输出：**[[],[0]]
```
?
```java
#### 方法一：迭代法实现子集枚举
class Solution {
    List<Integer> t = new ArrayList<Integer>();
    List<List<Integer>> ans = new ArrayList<List<Integer>>();

    public List<List<Integer>> subsets(int[] nums) {
        int n = nums.length;
        for (int mask = 0; mask < (1 << n); ++mask) {
            t.clear();
            for (int i = 0; i < n; ++i) {
                if ((mask & (1 << i)) != 0) {
                    t.add(nums[i]);
                }
            }
            ans.add(new ArrayList<Integer>(t));
        }
        return ans;
    }
}
复杂度分析
时间复杂度：O(n×2^n)。一共 2^n个状态，每种状态需要 O(n) 的时间来构造子集。
空间复杂度：O(n)。即构造子集使用的临时数组 t 的空间代价。

#### 方法二：递归法实现子集枚举。
class Solution {
    List<Integer> t = new ArrayList<Integer>();
    List<List<Integer>> ans = new ArrayList<List<Integer>>();

    public List<List<Integer>> subsets(int[] nums) {
        dfs(0, nums);
        return ans;
    }

    public void dfs(int cur, int[] nums) {
        if (cur == nums.length) {
            ans.add(new ArrayList<Integer>(t));
            return;
        }
        t.add(nums[cur]);
        dfs(cur + 1, nums);
        t.remove(t.size() - 1);
        dfs(cur + 1, nums);
    }
}

复杂度分析
时间复杂度：O(n×2^n)。一共 2^n个状态，每种状态需要 O(n) 的时间来构造子集。
空间复杂度：O(n)。临时数组 tt 的空间代价是 O(n)，递归时栈空间的代价为 O(n)。
```
<!--SR:!2024-10-12,4,210-->

```java
#### [90. 子集 II](https://leetcode.cn/problems/subsets-ii/)

难度中等940

给你一个整数数组 `nums` ，其中可能包含重复元素，请你返回该数组所有可能的子集（幂集）。

解集 **不能** 包含重复的子集。返回的解集中，子集可以按 **任意顺序** 排列。

**示例 1：**

**输入：**nums = [1,2,2]
**输出：**[[],[1],[1,2],[1,2,2],[2],[2,2]]

**示例 2：**

**输入：**nums = [0]
**输出：**[[],[0]]

```
?
```java
#### 方法一：迭代法实现子集枚举
思路

考虑数组 [1, 2, 2][1,2,2]，选择前两个数，或者第一、三个数，都会得到相同的子集。

也就是说，对于当前选择的数 xx，若前面有与其相同的数 yy，且没有选择 yy，此时包含 xx 的子集，必然会出现在包含 yy 的所有子集中。

我们可以通过判断这种情况，来避免生成重复的子集。代码实现时，可以先将数组排序；迭代时，若发现没有选择上一个数，且当前数字与上一个数相同，则可以跳过当前生成的子集。

class Solution {
    List<Integer> t = new ArrayList<Integer>();
    List<List<Integer>> ans = new ArrayList<List<Integer>>();

    public List<List<Integer>> subsetsWithDup(int[] nums) {
        Arrays.sort(nums);
        int n = nums.length;
        for (int mask = 0; mask < (1 << n); ++mask) {
            t.clear();
            boolean flag = true;
            for (int i = 0; i < n; ++i) {
                if ((mask & (1 << i)) != 0) {
                    if (i > 0 && (mask >> (i - 1) & 1) == 0 && nums[i] == nums[i - 1]) {
                        flag = false;
                        break;
                    }
                    t.add(nums[i]);
                }
            }
            if (flag) {
                ans.add(new ArrayList<Integer>(t));
            }
        }
        return ans;
    }
}
// if (mask & (1 << i))这句判断mask从右往左第i位是不是1，是1则选择了数组从右往左第i个数；如果(mask >> (i - 1) & 1) == 0，说明mask从右往左第i-1位为0，未选择数组从右往左的第i-1个数。第i个数和第i-1个数相同时，未选择第i-1个只选择第i个一定会重复，所以break。

复杂度分析

时间复杂度：O(n×2^n)，其中 nn 是数组 nums 的长度。排序的时间复杂度为 O(nlogn)。一共 2^n个状态，每种状态需要 O(n) 的时间来构造子集，一共需要 O(n×2^n) 的时间来构造子集。由于在渐进意义上 O(nlogn) 小于 O(n×2^n)，故总的时间复杂度为 O(n×2^n)。

空间复杂度：O(n)O(n)。即构造子集使用的临时数组 tt 的空间代价。
#### 方法二：递归法实现子集枚举
class Solution {
    List<Integer> t = new ArrayList<Integer>();
    List<List<Integer>> ans = new ArrayList<List<Integer>>();

    public List<List<Integer>> subsetsWithDup(int[] nums) {
        Arrays.sort(nums);
        dfs(false, 0, nums);
        return ans;
    }

    public void dfs(boolean choosePre, int cur, int[] nums) {
        if (cur == nums.length) {
            ans.add(new ArrayList<Integer>(t));
            return;
        }
        dfs(false, cur + 1, nums);
        if (!choosePre && cur > 0 && nums[cur - 1] == nums[cur]) {
            return;
        }
        t.add(nums[cur]);
        dfs(true, cur + 1, nums);
        t.remove(t.size() - 1);
    }
}
复杂度分析

时间复杂度：O(n×2^n)，其中 nn 是数组nums 的长度。排序的时间复杂度为 O(nlogn)。最坏情况下 nums 中无重复元素，需要枚举其所有 2^n个子集，每个子集加入答案时需要拷贝一份，耗时 O(n)，一共需要 O(n×2^n)+O(n)=O(n×2^n) 的时间来构造子集。由于在渐进意义上 O(nlogn) 小于 O(n×2^n)，故总的时间复杂度为 O(n×2^n)。

空间复杂度：O(n)O(n)。临时数组 t 的空间代价是 O(n)O(n)，递归时栈空间的代价为 O(n)O(n)。


```
<!--SR:!2024-10-12,4,210-->

```java
#### [剑指 Offer 32 - II. 从上到下打印二叉树 II](https://leetcode.cn/problems/cong-shang-dao-xia-da-yin-er-cha-shu-ii-lcof/)

从上到下按层打印二叉树，同一层的节点按从左到右的顺序打印，每一层打印到一行。
```
?
```java
class Solution {
    public List<List<Integer>> levelOrder(TreeNode root) {
        List<List<Integer>> ret = new ArrayList<List<Integer>>();
        if (root == null) {
            return ret;
        }

        Queue<TreeNode> queue = new LinkedList<TreeNode>();
        queue.offer(root);
        while (!queue.isEmpty()) {
            List<Integer> level = new ArrayList<Integer>();
            int currentLevelSize = queue.size();
            for (int i = 1; i <= currentLevelSize; ++i) {
                TreeNode node = queue.poll();
                level.add(node.val);
                if (node.left != null) {
                    queue.offer(node.left);
                }
                if (node.right != null) {
                    queue.offer(node.right);
                }
            }
            ret.add(level);
        }
        
        return ret;
    }
}
复杂度分析
记树上所有节点的个数为 n。
时间复杂度：每个点进队出队各一次，故渐进时间复杂度为 O(n)。
空间复杂度：队列中元素的个数不超过 nn 个，故渐进空间复杂度为 O(n)。
```
<!--SR:!2024-01-22,4,210-->

```java
#### [剑指 Offer 32 - III. 从上到下打印二叉树 III](https://leetcode.cn/problems/cong-shang-dao-xia-da-yin-er-cha-shu-iii-lcof/)


请实现一个函数按照之字形顺序打印二叉树，即第一行按照从左到右的顺序打印，第二层按照从右到左的顺序打印，第三行再按照从左到右的顺序打印，其他行以此类推。
```
?
```java
class Solution {
    public List<List<Integer>> zigzagLevelOrder(TreeNode root) {
        List<List<Integer>> ans = new LinkedList<List<Integer>>();
        if (root == null) {
            return ans;
        }

        Queue<TreeNode> nodeQueue = new ArrayDeque<TreeNode>();
        nodeQueue.offer(root);
        boolean isOrderLeft = true;

        while (!nodeQueue.isEmpty()) {
            Deque<Integer> levelList = new LinkedList<Integer>();
            int size = nodeQueue.size();
            for (int i = 0; i < size; ++i) {
                TreeNode curNode = nodeQueue.poll();
                if (isOrderLeft) {
                    levelList.offerLast(curNode.val);
                } else {
                    levelList.offerFirst(curNode.val);
                }
                if (curNode.left != null) {
                    nodeQueue.offer(curNode.left);
                }
                if (curNode.right != null) {
                    nodeQueue.offer(curNode.right);
                }
            }
            ans.add(new LinkedList<Integer>(levelList));
            isOrderLeft = !isOrderLeft;
        }

        return ans;
    }
}

复杂度分析

时间复杂度：O(N)，其中 N 为二叉树的节点数。每个节点会且仅会被遍历一次。
空间复杂度：O(N)。我们需要维护存储节点的队列和存储节点值的双端队列，空间复杂度为 O(N)。
```
<!--SR:!2024-01-22,4,210-->