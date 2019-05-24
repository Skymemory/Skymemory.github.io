---
layout:     post
title:      "String And Binary Search Of LeetCode"
date:       2019-05-22 15:00:00 +0800
author:     "Sky丶Memory"
header-img: "img/2019-05-22-01-bg.jpeg"
tags: Algorithm LeetCode String BinarySearch
usemathjax: true

---

整理了一些*LeetCode*上关于字符串、二分搜索专题的一些题目。

---

#### [Word Pattern](https://leetcode.com/problems/word-pattern/)

给定字符串*letters*、*words*，建立*letter*与*word*之间映射，使得*letters*与*words*根据映射表可以互换。

解题代码:

```python
class Solution:
    def wordPattern(self, pattern: str, str: str) -> bool:
        words = str.split()
        if len(words) != len(pattern):
            return False
        letter2word = dict()
        word2letter = dict()
        for letter, word in zip(pattern, words):
            had = letter2word.get(letter)
            # already allocated but not equal
            if had and had != word:
                return False
            letter2word[letter] = word
            
            # check validness
            had = word2letter.get(word)
            if had and had != letter:
                return False
            word2letter[word] = letter
        
        return True
```

---

#### [Find K-th Smallest Pair Distance](https://leetcode.com/problems/find-k-th-smallest-pair-distance/)

任选两个数字的组合数为$C_n^2$，可以考虑将问题转换为$m$个有序数组中查找排名第$k$大的元素，这样就能通过二分枚举第$k$大元素来解决，整体时间复杂度为$O(n* \log n+n* \log w)$，其中$w$为距离对值域范围。

解题代码:

```python
class Solution:
    def smallestDistancePair(self, nums: List[int], k: int) -> int:
        order = sorted(nums)

        low, high = 0, order[-1] - order[0]
        while low < high:
            mid = (low + high) >> 1
            rank = self.get_rank(order, mid)
            if rank >= k:
                high = mid
            else:
                low = mid + 1
        return low

    @staticmethod
    def get_rank(nums, possible):
        cnt = j = 0
        for i, v in enumerate(nums):
            while j < len(nums) and nums[j] - v <= possible:
                j += 1
            cnt += j - i - 1
        return cnt
```

---

#### [Split Array Largest Sum](https://leetcode.com/problems/split-array-largest-sum/)

经典的二分题目。。。。

解题思路:

```python
class Solution:
    @staticmethod
    def check(nums, m, at_most):
        slide, cnt = at_most + 1, 0
        for elem in nums:
            if elem > at_most:
                return False
            if slide + elem <= at_most:
                slide += elem
            else:
                slide = elem
                cnt += 1
        return cnt <= m
    
    def splitArray(self, nums: List[int], m: int) -> int:
        low, high = 0, sum(nums)
        while low < high:
            mid = (low + high) >> 1
            ok = self.check(nums, m, mid)
            if ok:
                high = mid
            else:
                low = mid + 1
        return high
```

