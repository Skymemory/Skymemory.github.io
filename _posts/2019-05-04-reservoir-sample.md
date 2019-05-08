---
layout:     post
title:      "蓄水池采样算法"
date:       2019-05-04 15:00:00 +0800
author:     "Sky丶Memory"
header-img: "img/2019-05-04-01-bg.jpg"
tags: Math BigData
usemathjax: true


---

蓄水池采样算法主要用于解决一类问题：输入样本空间未知的情况下，抽取$k$个样本，使得每个样本被选中的概率相同。

算法步骤，针对第$n$个样本$X_n$：

- $ n <= k $：$X_n$直接放入池中
- $n > k$：以$\frac k n$的概率选中样本$X_n$，若选中，则从池中随机选取样本$X_i$，用$X_n$替换$X_i$

#### 证明

首先，考察第$j(j > k)$个样本$X_j$，$P(被X_j替换)=\frac k j * \frac 1 k=\frac1j$，故而得出

$$P(不被X_j替换) = 1 - P(被X_j替换) = \frac {j-1}j$$

求每个样本被选中的概率分两种情况：

1. 第$i(i \leq k)$个样本$X_i$被选中的概率

   $$P(X_i被选中)=P(不被X_{k+1}替换) * P(不被X_{k+2}替换)…*P(不被X_n替换)=\frac k{k+1}*\frac{k+1}{k+2}…* \frac {n-1}{n}=\frac kn$$

2. 第$i(i > k)$个样本$X_i$被选中的概率

   $$P(X_i被选中) = \frac k i * P(不被X_{i+1}替换) * P(不被X_{i+2}替换)… * P(不被X_n替换)=\frac k i * \frac i {i+1} * \frac {i+1}{i+2}…\frac {n-1}n=\frac k n$$

根据上述的证明可以发现，最终每个样本被选中的概率都为$\frac kn$。

---

#### 实现

```python
import random
import unittest
from collections import Counter


def reservoir_sample(stream, k):
    n = 0
    it = iter(stream)
    pool = []
    for item in it:
        n += 1
        if n <= k:
            pool.append(item)
        elif k > 0:
            rr = random.randint(0, n - 1)
            if rr < k:
                pool[rr] = item
    return pool


class TestMain(unittest.TestCase):
    def test_reservoir_sample(self):
        stat = Counter()
        for i in range(100000):
            outcome = reservoir_sample(range(10), 3)
            stat.update(outcome)
        print(stat)


if __name__ == '__main__':
    unittest.main()

```

根据概率的统计定义，重复实验的次数越来越多时，得出的概率模型和已知概率模型应当具有一致性。上述代码中，通过$10$万次的蓄水池采样，可以发现不同数字出现的次数都在3万左右，从一定意义上也验证了蓄水池采样算法的正确性。