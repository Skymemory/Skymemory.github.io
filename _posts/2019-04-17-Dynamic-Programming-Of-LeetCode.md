---

layout:     post
title:      "Dynamic Programming Of LeetCode"
date:       2019-04-17 15:00:00 +0800
author:     "Sky丶Memory"
header-img: "img/2019-04-17-01-bg.jpg"
tags: Algorithm LeetCode DP
usemathjax: true

---

整理了一些*LeetCode*上比较有意思或当时没思路的题目，其中大部分都是*DP*。

#### [Edit Distance](<https://leetcode.com/problems/edit-distance/>)

定义$dp[i+1][j+1]$表示*s[0…i]*、*t[0…j]*最短编辑距离，考察*s[i]*、*t[j]*：

- 若$s[i] = t[j]，dp[i+1][j+1] = dp[i][j]$
- 若$s[i] \neq t[j]$，分三种情况
  - 替换：$dp[i+1][j+1] = dp[i][j]  +  1$
  - 插入：$dp[i+1][j+1] = dp[i+1][j]  +  1$
  - 删除：$dp[i+1][j+1] = dp[i][j+1]  +  1$

根据上述分析，可得出状态转移方程：

$$dp[i+1][j+1]=\begin{cases}

dp[i][j] & s[i] = t[j] \\

\min(dp[i][j], dp[i][j+1], dp[i+1][j]) + 1 & s[i] \neq t[j]

\end{cases}$$

解题代码：

```python
class Solution:
    def minDistance(self, word1: str, word2: str) -> int:
        l1, l2 = len(word1), len(word2)
        dp = [[0] * (l2 + 1) for _ in range(l1 + 1)]
        
        for j in range(l2 + 1):
            dp[0][j] = j
        
        for i in range(l1 + 1):
            dp[i][0] = i
        
        for i in range(l1):
            for j in range(l2):
                if word1[i] == word2[j]:
                    dp[i + 1][j + 1] = dp[i][j]
                else:
                    dp[i + 1][j + 1] = min(dp[i][j], dp[i][j + 1], dp[i + 1][j]) + 1
        
        return dp[l1][l2]
```

---

#### [Unique Binary Search Trees](<https://leetcode.com/problems/unique-binary-search-trees/>)

完全的思维题目，思维方向不对的话，感觉很难做出来。

*dp[i\]*表示*i*个节点的不同二叉搜索树个数，通过枚举可能的根节点，累计计数即可。比如当$n=5$时，以*2*为根节点的不同二叉搜索树个数为 $dp[1] * dp[3]$（根节点为*2*左支节点个数为*1*，右支节点个数为*3*），这样重叠子问题的性质就体现出来了，状态转移方程为：

$$dp[i] = \begin{cases}

1 & i = 0,1\\

\sum_{j=0}^{i-1}  dp[j] * dp[i - j - 1] & i \geq 2

\end{cases}$$

解题代码:

```python
class Solution:
    def numTrees(self, n: int) -> int:
        dp = [1 for _ in range(n + 1)]
        for i in range(2, n + 1):
            dp[i] = 0
            for j in range(i):
                dp[i] += dp[j] * dp[i - j - 1]
        return dp[n]
```

---

#### [Largest Rectangle in Histogram](<https://leetcode.com/problems/largest-rectangle-in-histogram/>)

解题的关键点在于最优矩形中存在一个性质：*假设最优矩形以i为起点、j为终点，那必然存在$heights[i] > heights[i - 1]、heights[j] > heights[j + 1]$*。

由该性质可以得出，矩形的右边界所在的位置必然是个降序点（两端点外的边界可看成是高度为*0*的直方图），找到可能的右边界点，据此计算已经能确定的左边界点，这样就能做到在*O(n)*时间范围内求解出结果。

解题代码:

```python
class Solution:
    def largestRectangleArea(self, heights: List[int]) -> int:
        ans = 0
        dq = collections.deque()
        for idx, h in enumerate(itertools.chain(heights, [0])):
            last = None
            while dq and dq[-1][1] >= h:
                left, n_h = last = dq.pop()
                ans = max(ans, (idx - left) * n_h)
            dq.append((last[0] if last else idx, h))
        return ans
```



---

####  [ Maximal Rectangle](<https://leetcode.com/problems/maximal-rectangle/>)

这道题目比较有意思，利用前缀和思想是很容易想到$O(n * m * \min(n, m))$，不过时间复杂度还是略微有点高。

最优解的做法在于将问题转换为另一个问题——[Largest Rectangle in Histogram](<https://leetcode.com/problems/largest-rectangle-in-histogram/>)，这样就能通过枚举不同行在$O(n * m)$时间复杂度范围内解决。

解题代码：

```python
class Solution:
    def maximalRectangle(self, matrix: List[List[str]]) -> int:
        if not matrix:
            return 0
        n, m = len(matrix), len(matrix[0])
        h = [0] * m
        ans = 0
        for i in range(n):
            for j in range(m):
                h[j] = 0 if matrix[i][j] == '0' else h[j] + 1
            ans = max(ans, self.largestRectangleArea(h))
        return ans
             
    def largestRectangleArea(self, heights: List[int]) -> int:
        ans = 0
        dq = collections.deque()
        for idx, h in enumerate(itertools.chain(heights, [0])):
            last = None
            while dq and dq[-1][1] >= h:
                left, n_h = last = dq.pop()
                ans = max(ans, (idx - left) * n_h)
            dq.append((last[0] if last else idx, h))
              
            
        return ans
```

---

#### [House Robber II](https://leetcode.com/problems/house-robber-ii/)

在环中相邻不能选这个约束条件，可以拆分成两个重叠的子问题：$h_1…h_{n-1}$和$h_2…h_n$，进而就能求出最优解。

解题代码:

```python
class Solution:
    def rob(self, nums: List[int]) -> int:
        n = len(nums)
        if n <= 1:
            return nums[0] if n else 0
        dp = [[0] * (n + 2) for _ in range(2)]
        for i in range(2):
            for j in range(i, n - 1 + i):
                dp[i][j + 2] = max(dp[i][j + 1], nums[j] + dp[i][j])
        return max(itertools.chain.from_iterable(dp))
```

---

#### [ Minimum Score Triangulation of Polygon](https://leetcode.com/problems/minimum-score-triangulation-of-polygon/)

*dp\[i]\[j]*表示沿顺时针方向，以*i*为起点、*j*为终点的多边形三角化最小值，通过枚举可能的分割点故而得到状态转移方程：

$$dp[i][j]=\min(dp[i][k] + A[i]*A[k]*A[j] + dp[k][j])，i < k < j $$

解题代码:

```python
class Solution:
    def minScoreTriangulation(self, A: List[int]) -> int:
        n = len(A)
        dp = [[0] * n for _ in range(n)]
        for j in range(3, n + 1):
            for i in range(0, n - j + 1):
                v_min = 10 ** 9 
                for k in range(i + 1, i + j - 1):
                    v_min = min(dp[i][k] + A[i] * A[k] * A[i + j - 1] + dp[k][i + j - 1],
                                v_min)
                dp[i][i + j - 1] = v_min
        return dp[0][n - 1]        
```



