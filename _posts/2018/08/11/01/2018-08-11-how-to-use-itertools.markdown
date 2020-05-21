---
layout:     post
title:      "itertools基本使用"
date:       2018-08-11 23:30:00 +0800
author:     "Sky丶Memory"
header-img: "img/2018/08/11/01/bg.jpg"
tags: Python
---

itertools算得上是Python中比较常用的一个模块，它主要提供了操作在可迭代对象上的一些实用函数，这篇文章主要介绍这些函数的基本等价实现与一些使用心得.(Python Version:3.6.3)

#### cycle
基本等价实现:
``` python
def cycle(iterable):
    # cycle('ABCD') ---> A B C D A B C D A B C D...
    saved = []
    for item in iterable:
        yield item
        saved.append(item)
    while saved:
        for item in saved:
            yield item
```
使用例子：基于cycle实现的Round-Robin负载均衡
``` python
class RoundRobinSelector:
    def __init__(self, hosts):
        self.hosts = hosts
        self._it = cycle(hosts)
    
    def select(self):
        return next(self._it)
```
之前去知乎面试，考过手写cycle的实现。。。。另外，cycle使用中有个地方需要注意，它需要额外的空间来存储，所以当输入集太大时，最好自己用取模实现类似的功能

---

#### count
基本等价实现:
``` python
def count(start=0, step=1):
    # count(10) ---> 0 1 2 3 4 5 6...
    # count(2.5, 0.5) ---> 2.5 3 3.5...
    n = start
    while True:
        yield n
        n += step
```

---

#### repeat
基本等价实现：
``` python
def repeat(o, times=None):
    if times is None:
        while 1:
            yield o
    else:
        for _ in range(times):
            yield o
```

使用例子：
    repeat函数主要用在一些需要常量序列的地方，比如对一个序列求平方---map(pow, range(5), repeat(2))
    
---

#### compress
基本等价实现:
``` python
def compress(iterable, selectors):
    # compress('ABCDED', [1, 0, 1, 0, 0, 0]) ---> A C
    return (d for d, s in zip(iterable, selectors) if s)
```
---

#### filterfalse
基本等价实现：
``` python
def filterfalse(pred, iterable):
    if pred is None:
        pred = bool
    return (item for item in iterable if not pred(item))

def filter(pred, iterable):
    if pred is None:
        pred = bool
    return (item for item in iterable if pred(item))
```

filterfalse配合内置函数filter可用于对输入集通过谓词(predicate)进行分组，对得到的序列可用于不同业务逻辑的处理

---

#### takewhile
基本等价实现：
``` python
def takewhile(pred, iterable):
    for item in iterable:
        if pred(item):
            yield item
        else:
            break
```

用于得到一个符合条件的前缀序列

---

#### dropwhile
基本等价实现:
``` python
def dropwhile(pred, iterable):
    # dropwhile(lambda x: x < 5, [1, 4, 6, 4, 1]) ---> 6 4 1
    it = iter(iterable)
    for item in it:
        if not pred(item):
            yield item
            break
    for item in it:
        yield item
```
用于得到一个符合条件的后缀序列

---

#### chain
基本等价实现:
``` python
def chain(*iterables):
    # chain('ABC', 'DEF') --> A B C D E F
    for it in iterables:
        for element in it:
            yield element
```
用于将连续的多个序列当成单个序列处理

---
#### chain.from_iterable
基本等价实现:
``` python
def from_iterable(iterables):
    # chain.from_iterable(['ABC', 'DEF']) --> A B C D E F
    for it in iterables:
        for element in it:
            yield element
```

跟chain类似，唯一的区别在于，chain.from_iterable不用事先知道iterables的长度，而chain则需要

---

#### islice

基本等价实现:
``` python
def islice(iterable, *args):
    # islice('ABCDEFG', 2) --> A B
    # islice('ABCDEFG', 2, 4) --> C D
    # islice('ABCDEFG', 2, None) --> C D E F G
    # islice('ABCDEFG', 0, None, 2) --> A C E G
    s = slice(*args)
    it = iter(range(s.start or 0, s.stop or sys.maxsize, s.step or 1))
    try:
        nexti = next(it)
    except StopIteration:
        return
    for i, element in enumerate(iterable):
        if i == nexti:
            yield element
            nexti = next(it)
```
该函数主要作用有如下两个：

- 从一定程度上避免了常规切片带来的内存复制问题，同时也带来了另外一个问题，切片的时间复杂度增为O(n)，使用时，需要自己权衡一下

- 提供了generator切片，但start、stop参数不支持负数，需注意该限制

---

#### accumulate
基本等价实现:
``` python
def accumulate(iterable, func=operator.add):
    'Return running totals'
    # accumulate([1,2,3,4,5]) --> 1 3 6 10 15
    # accumulate([1,2,3,4,5], operator.mul) --> 1 2 6 24 120
    it = iter(iterable)
    try:
        total = next(it)
    except StopIteration:
        return
    yield total
    for element in it:
        total = func(total, element)
        yield total
```

该函数与functools.reduce类似，唯一区别在于，functools.reduce只返回最终计算的值，而accumulate返回了每步计算的值，通过accumulate函数，我们很容易得出前缀相关的性质，如前缀和、前缀积、前缀最值等

---

#### zip_longest
基本等价实现:
``` python
class ZipExhausted(Exception):
    pass

def zip_longest(*args, **kwds):
    # zip_longest('ABCD', 'xy', fillvalue='-') --> ('A', 'x'), ('B', 'y'), ('C', '-'), ('D', '-')
    fillvalue = kwds.get('fillvalue')
    counter = len(args) - 1
    def sentinel():
        nonlocal counter
        if not counter:
            raise ZipExhausted
        counter -= 1
        yield fillvalue
    fillers = repeat(fillvalue)
    iterators = [chain(it, sentinel(), fillers) for it in args]
    try:
        while iterators:
            yield tuple(map(next, iterators))
    except ZipExhausted:
        pass
```
内置函数zip当参数序列长度不等时，选择最短序列的长度做为最终结果的长度，而zip_longest刚好与其相反，选择最长序列长度做为最终结果长度，而不及最长长度的序列，可通过fillvalue指定填充值

---

#### groupby

基本等价实现:
``` python
class groupby:
    # [k for k, g in groupby('AAAABBBCCDAABBB')] --> A B C D A B
    # [list(g) for k, g in groupby('AAAABBBCCD')] --> AAAA BBB CC D
    def __init__(self, iterable, key=None):
        if key is None:
            key = lambda x: x
        self.keyfunc = key
        self.it = iter(iterable)
        self.tgtkey = self.currkey = self.currvalue = object()
    def __iter__(self):
        return self
    def __next__(self):
        while self.currkey == self.tgtkey:
            self.currvalue = next(self.it)    # Exit on StopIteration
            self.currkey = self.keyfunc(self.currvalue)
        self.tgtkey = self.currkey
        return (self.currkey, self._grouper(self.tgtkey))
    def _grouper(self, tgtkey):
        while self.currkey == tgtkey:
            yield self.currvalue
            try:
                self.currvalue = next(self.it)
            except StopIteration:
                return
            self.currkey = self.keyfunc(self.currvalue)
```

通过指定key函数，对可迭代对象进行分组，分组的依据为相邻元素是否相等来决定是否属于同一个组，这与SQL中的Group By处理方式不同，使用时需要额外注意


#### 参考资料

- [itertools](https://docs.python.org/3.6/library/itertools.html)