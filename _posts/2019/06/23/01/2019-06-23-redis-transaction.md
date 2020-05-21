---
layout:     post
title:      "Redis事务"
date:       2019-06-23 20:00:00 +0800
author:     "Sky丶Memory"
header-img: "img/2019/06/23/01/bg.jpg"
tags: Redis Storage
usemathjax: true

---

最近看`redis-py`源代码时，发现其支持`Redis`自身事务特性，就顺带整理了下相关的知识，这里做简述记录。

`Redis`事务与传统关系型数据库中的事务略有不同，它的事务提供两个保障：

- 事务中的命令顺序执行，并且在执行当前事务期间，不会有其他客户端命令穿插进来，这点与它提供的`pipeline`功能存在差异：`pipeline`中的命令是有可能穿插其他客户端的命令混合执行的。
- 事务中的命令要么全部执行要么都不执行，但这里面有些特殊场景，当使用`AOF`持久化方式时，`Redis`事务有可能会出现部分执行状态，比如说事务执行期间断电、人为`shutdown`等，`Redis`并没有采用类似`log`写成功后才执行事务方式，故而有可能导致事务执行期间挂掉恢复后状态不一致，这点可以通过`redis-check-aof`命令来检查。

#### 错误处理机制

事务中的命令会放到类似队列的数据结构中，直到客户端发送`exec`命令，此时服务端顺序执行队列中的命令。

`Redis`会在命令入队时，做一定的前置检查(比如参数是否正确等)，此时若发现命令有问题，会返回客户端错误。

`Redis`在事务执行期间，若发现某条命令有问题会记录对应命令执行的错误信息，然后继续执行剩下的命令。

#### WATCH

`Redis`除了提供基本事务操作命令`MULTI`、`DISCARD`、`EXEC`外，还提供了另外一个事务相关的命令——`WATCH`，`WATCH`主要作用在于配合`Redis`事务实现类似`CAS`逻辑。

比如，`incr`命令基于`WATCH`实现方式:

````shell
WATCH mykey
val = GET mykey
val = val + 1
MULTI
SET mykey $val
EXEC
````

`WATCH`会监听指定键变更操作，然后在事务执行期间根据是否产生变更来决定事务是否执行。

基于`Python`客户端实现`INCR`命令:

```python
import redis
from redis import WatchError

r = redis.StrictRedis()
name = 'counter'
with r.pipeline(transaction=True) as pipe:
    while 1:
        try:
            # startup immediately execute mode.
            pipe.watch(name)
            old = pipe.get(name)
            new = int(old) + 1
            pipe.multi()
            pipe.set(name, new)
            pipe.execute()
            break
        except WatchError:
            continue
```

