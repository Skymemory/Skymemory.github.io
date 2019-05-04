---
layout:     post
title:      "Python之迭代器协议"
date:       2019-05-04 21:00:00 +0800
author:     "Sky丶Memory"
header-img: "img/2019-05-04-02-bg.jpg"
tags: Python

---

Python中常见的迭代形式为*for…in…*，其背后对应的正是迭代器协议。

迭代器协议需要支持两个魔方方法：*\_\_iter\_\_*和*\_\_next\_\_*，实现了迭代器协议的对象即为迭代器。

其中，迭代器、可迭代、生成器、容器之间的关系图：

![](/img/2019-05-04-02-01.png)

Python提供了两个内建函数操作迭代器：*iter*和*next*，前者用于将对象转换为迭代器，后者用于从迭代器中获取值。

*iter*有种鲜为人知的使用方式：

```python
from functools import partial
with open('mydata.db', 'rb') as f:
    for block in iter(partial(f.read, 64), b''):
        process_block(block)
```

在这种使用方式中，迭代器是临时创建的，其*\_\_next\_\_*方法由传入的两个参数决定。