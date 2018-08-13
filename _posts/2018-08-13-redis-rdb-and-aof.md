---
layout:     post
title:      "Redis持久化"
date:       2018-08-13 23:30:00 +0800
author:     "Sky丶Memory"
header-img: "img/2018-08-13-01-bg.webp"
catalog: true
tags:
    - redis
    - 缓存
    - RDB
    - AOF
---


Redis为持久化提供了两种方式:

- RDB: 定期对Server中的数据集进行快照(Snapshot)备份，以全量方式进行

- AOF: 记录下每次对Server中的写操作，类似增量备份思想

本文主要介绍这两种持久化方式的使用及基本原理，以便加深自己的理解.

---

## RDB

#### RDB配置选项

Redis默认以RDB方式定期对Server中的数据集进行快照备份，其持久化配置有如下选项:

``` text
# 备份目录
dir ./

# 备份文件名
dbfilename dump.rdb

# 指定备份策略，也就是何时进行备份
save <seconds>  <changes>

# 如果备份出错，Server是否停止写操作
stop-writes-on-bgsave-error yes

# 写入.rdb文件时，是否对字符串对象开启LZF压缩
rdbcompression yes

# 导入时是否检查
rdbchecksum yes
```

配置本身很简单，这里只解释几个重要选项:

- save n m:指定在时间间隔n秒内，主进程产生了至少m次写操作，则进行快照备份 ;该选项可重复配置

- stop-writes-on-bgsave-error:该选项主要用于控制备份出错时，是否停止Server的写操作；默认情况下，值为yes，也就是说如果备份出错，Server变成只读，同时需要注意的是，在该情况下，修复好备份出现的问题后，需要手动调用bgsave或save方可让Server可写

另外，如果需要禁用RDB方式，只需在save的最后一行下面添加save ""新行即可

#### RDB原理

RDB触发方式分为两种：手动、自动

- 手动触发
    - save: 阻塞当前Server，直到备份完成，其间Server无法处理其他客户端的请求
    - bgsave: 由Server fork出一个子进程来处理备份，所以阻塞只会出现在fork的时候

- 自动触发

    - 根据我们配置的save n m自动触发
    - 还有其他的自动触发条件，后面遇到在补充
    
RDB具体的备份逻辑是将当前Server中的数据集写入一个临时文件中，待写入完成时，将旧有的RDB文件替换，大致流程如下:
![](/img/2018-08-13-01-02.jpeg)

---

## AOF

#### AOF配置选项

AOF的运作方式和RDB有很大的区别，AOF采用记录每次写操作的方式进行数据的持久化，这里面主要需要考虑两个问题:

- 记录每次写操作的数据以什么方式落到到硬盘上

- 由于AOF记录的是**每次写操作**，这样在一些情况下会导致记录文件增长过大，比如在web网站中，一般我们会通过incr来记录每次PV、UV的增加，假设某天PV从0增加到了100W，那么，AOF记录文件就会至少存在100W行，这是很难接受的，会导致AOF记录文件占用过多的硬盘空间。不过，Redis提供了对应的解决方案---重写(rewrite)机制

下面先看下Redis提供的针对AOF持久化配置选项

``` text
# 是否开启AOF持久化
appendonly no

# 指定AOF文件名称
appendfilename "appendonly.aof"

# 数据同步到硬盘方式
appendfsync everysec

# 增长百分比设置
auto-aof-rewrite-percentage 100

# 触发重写时，文件大小至少达到多少
auto-aof-rewrite-min-size 64mb

# AOF文件不完整时是否截断
aof-load-truncated yes

# aof重写期间是否同步
no-appendfsync-on-rewrite no

# 文件重写策略
aof-rewrite-incremental-fsync yes


```

这里对上面一些重要选项进行说明:

appendfsync有三种模式:

- always: 把每个写操作立即同步到硬盘
- everysec: 每秒同步一次数据到硬盘
- no: 由OS来同步数据到硬盘，具体跟OS缓存页刷新有关
    
真实场景中，一般选用everysec方式，这也是Redis官方推荐的一种方式，具体的还是看业务需求


auto-aof-rewrite-percentage 指定增长百分比，用于对比当前AOF文件大小距离上次重写文件大小之间的增长百分比，如果当前文件的增加百分比大于该配置，则触发重写机制。指定为0，则禁用重写机制


#### AOF 原理

AOF持久化主要做两件事情，记录每次写操作和对AOF文件的重写

记录每次写操作的主要流程为: 写操作--->AOF_BUF--->硬盘，中间的AOF_BUF主要用于提供I/O性能

AOF文件的重写有手动触发、自动触发两种。

手动触发可通过调用命令BGREWRITEAOF

自动触发配置规则见上述配置选项，其具体的流程图大致如下所示:
![](/img/2018-08-13-01-01.jpeg)

对于上图中有几个关键点补充一下:

- 在重写期间，Server依旧会继续向旧AOF文件中写入数据，可用于保证如果重写失败，数据不会出现丢失
- fork出的子进程，重写数据是根据**Server当前的数据集，不是对旧AOF文件的compress**
- 重写期间，Server会额外维护一个aof_rewirte_buf用于记录fork子进程后的写操作，待子进程重写完成后，由Server将这部分增量追加到新的AOF临时文件中，这里面不会出现脏数据，主要原因还是归功于**Copy-On-Write机制**

---

## 番外

刚开始看RDB和AOF持久化方式时，一直有个问题没想明白，Server在可写的情况下，fork出的子进程读数据是怎么保证不脏读？
直到后面从官方文档中的一句话中才恍然大悟
> This method allows Redis to benefit from copy-on-write semantics.

COW，妙哉～


## 参考资料

 - [一文看懂Redis的持久化原理](https://juejin.im/post/5b70dfcf518825610f1f5c16)
 - [Redis Persistence](https://redis.io/topics/persistence)