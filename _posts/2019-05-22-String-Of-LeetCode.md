---
layout:     post
title:      "String Of LeetCode"
date:       2019-05-22 15:00:00 +0800
author:     "Sky丶Memory"
header-img: "img/2019-05-22-01-bg.jpeg"
tags: Algorithm LeetCode String
usemathjax: true

---

整理了一些*LeetCode*上关于字符串专题的一些题目。

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

