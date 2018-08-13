---
layout:     post
title:      "Redis 过期键删除策略"
date:       2018-08-13 11:30:00 +0800
author:     "Sky丶Memory"
header-img: "img/post-bg-2018-08-13.jpg"
catalog: true
tags:
    - redis
    - 缓存
---

在Redis中，我们可以通过为键设置过期时间，使其在一段时间范围内访问有效；而最近看Redis LRU时，突然想了解Redis是怎么处理过期键的删除这个问题，故而有此文以录之.

### 使用
Redis主要提供了[EXPIRE](https://redis.io/commands/expire)、[PEXPIRE](https://redis.io/commands/pexpire)、[EXPIREAT](https://redis.io/commands/expireat)、[PEXPIREAT](https://redis.io/commands/pexpireat)这四个基本命令给存在的键设置过期时间，时间参数支持相对时间和绝对Unix时间戳，几个简单例子：
``` text
> SET name SkyMemory
OK

> EXPIRE name 1                   # 1S后过期
(integer) 1

> PEXPIRE name 1000                # 1S后过期
(integer) 1

> EXPIREAT name 1533657600         # 2018-08-08 00:00:00过期
(integer) 1

> PEXPIREAT name 1533657600000     # 2018-08-08 00:00:00过期
```

通过TTL和PTTL命令可以获取到给定键距离过期还剩多少时间

---

### 内部机制

在Redis内部，存储键的过期时间采用统一的形式：精确到毫秒的Unix时间戳，这也就意味着，即便某台Redis Server挂了，等从AOF、RDB文件中恢复过来，关于设置的键过期信息依然有效。

当然，这样做也有些其他问题，比如时钟回拔、从其他服务器拷贝RDB或AOF文件恢复本机遇上的时间差异较大问题，这里暂不讨论时间差异问题

既然Redis内部会为每个设置过期时间的键记录对应的过期时间，那么，它是如何发现过期键并删除的呢？

Redis内部对过期键的删除分两种形式：主动、被动

- 被动:又称惰性删除，当客户端尝试获取某个键时，Redis发现该键已过期，则删除该键，并返回客户端不存在;当然，紧紧这么做是不够的，因为我们把键删除的行为依赖了客户端的访问，假想一种情况，大量的键在设置过期时间后，后续并没有客户端继续访问这些键，这样就会造成原本应该过期的键依然占据内存，很容易导致Redis出现OOM问题

- 主动：Redis内部会周期性的检测设置过期时间的键集合，从中随机删除过期键，具体步骤如下:
    - 随机从关联过期时间的键集合中选20个键
    - 删除这些键中已过期的键
    - 如果这20个键超过25%已过期，则重复执行步骤一

从上述中可以看出，主动方式用于解决被动方式不足，通过一定的采样及对样本的观察来决定整个过期键集合的分布，有一定的概率问题，至于主动检测方式的频率可通过Redis配置文件中的hz选项控制

另外，Redis内部对过期键的删除是采用了上述两种形式，这两个策略相互配合,可以很好地在合理利用 CPU时间和节约内存空间之间取得平衡

### 参考资料

- [Expire](https://redis.io/commands/expire)
    

