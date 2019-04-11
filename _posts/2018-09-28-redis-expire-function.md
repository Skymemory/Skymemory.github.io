---
layout:     post
title:      "Redis过期功能介绍"
date:       2018-09-28 19:30:00 +0800
author:     "Sky丶Memory"
header-img: "img/2018-09-28-01-bg.jpeg"
tags:
    - Redis
    - 存储
---

Redis作为一款NoSQL数据库产品，其自身提供了自动过期功能来清理失效数据，这可以大大简化开发人员的工作，使其更关注于业务层的逻辑，这篇文章从使用、自动清理策略、对持久化、复制的影响三个方面来介绍Redis过期功能。


#### 使用

Redis默认提供了5种方式设置生存或过期时间，其基本语法大体如下：

- EXPIRE key seconds: 将键key的生存时间设置为seconds秒

- EXPIREAT key timestamp: 将键key的过期时间设置为timestamp所指定的秒数时间戳

- PEXPIRE key milliseconds: 将键key的生存时间设置为milliseconds毫秒

- PEXPIREAT key milliseconds-timestamp：将键key的过期时间设置为milliseconds-timestamp所指定的毫秒时间戳

- SET key value [expiration EX seconds \| PX milliseconds]: 将键key的生存时间设置为seconds秒或milliseconds毫秒

简化一下上述方法，可以发现设置过期或生存时间无非相对和绝对两种时间方式，绝对时间戳可通过time命令获取Redis服务器当前时间戳，再根据具体的业务逻辑算出对应的时间戳即可。

Redis虽然提供了5种设置时间的方式，其内部设置方式却通过PEXPIREAT实现，其他的命令皆可间接转换为PEXPIREAT，具体转换逻辑伪代码如下所示：

```python
def EXPIRE(key,ttl_in_sec):

    #将TTL从秒转换成毫秒
    ttl_in_ms = sec_to_ms(ttl_in_sec)

    PEXPIRE(key, ttl_in_ms)

def PEXPIRE(key,ttl_in_ms):

    #获取以毫秒计算的当前UNIX时间戳
    now_ms = get_current_unix_timestamp_in_ms()

    #当前时间加上TTL，得出毫秒格式的键过期时间
    PEXPIREAT(key,now_ms+ttl_in_ms)

def EXPIREAT(key,expire_time_in_sec):

    # 将过期时间从秒转换为毫秒
    expire_time_in_ms = sec_to_ms(expire_time_in_sec)

    PEXPIREAT(key, expire_time_in_ms)

```

键设置过期时间后，可通过TTL或PTTL查看键的剩余生存时间：
```powershell
redis> set name SkyMemory
OK

redis> expire name 20
(integer) 1

redis> ttl name
(integer) 15

# 20s后
redis> ttl name 
(integer) -2
```

同样地，可以通过PERSIST命令显示删除键的过期时间：

```powershell
redis> set name skymemory ex 120
OK

redis> ttl name
(integer) 118

redis> PERSIST name
(integer) 1

redis> ttl name
(integer) -1

```


#### 过期删除策略

通过指定键过期时间后，Redis默认会自动清理失效的键，那Redis具体的清理策略是什么呢？

Redis内部记录键过期时间格式为绝对Unix时间戳，精确到毫秒，对于过期键的删除策略，Redis内部有２种形式：惰性删除和定时删除

惰性删除，又称被动删除，基本思路大概就是将对键的删除操作延迟到对键的访问，也就是每次对键进行操作时，判断键是否过期：
![](/img/2018-09-28-01-01.jpeg)


这种方式对CPU来说是友好的，尽量不占用CPU资源，但对内存来说，又是不友好的，因为键的删除依赖客户端的访问，
如果客户端设置了大量键的过期时间，在过期时间到达后，并没有再访问Redis服务器，这些键会一直占用内存，最终导致Redis服务器内存资源不够，
为了避免这种情况的发生，Redis又额外的增加了定时删除的策略来尽量保证不会出现这种情况。


定时删除，又称主动删除，基本思路大概就是服务器定期(默认1S运行10次)运行删除过期键的任务来清理失效键，整个过程的伪代码描述如下：
```python
# 默认每次检查的数据库数量
DEFAULT_DB_NUMBERS = 16

# 默认每个数据库检查的键数量
DEFAULT_KEY_NUMBERS = 20

# 全局变量，记录检查进度
current_db = 0

def activeExpireCycle():

    # 初始化要检查的数据库数量
    # 如果服务器的数据库数量比 DEFAULT_DB_NUMBERS 要小
    # 那么以服务器的数据库数量为准
    if server.dbnum < DEFAULT_DB_NUMBERS:
        db_numbers = server.dbnum
    else:
        db_numbers = DEFAULT_DB_NUMBERS

    # 遍历各个数据库
    for i in range(db_numbers):

        # 如果current_db的值等于服务器的数据库数量
        # 这表示检查程序已经遍历了服务器的所有数据库一次
        # 将current_db重置为0，开始新的一轮遍历
        if current_db == server.dbnum:
            current_db = 0

        # 获取当前要处理的数据库
        redisDb = server.db[current_db]

        # 将数据库索引增1，指向下一个要处理的数据库
        current_db += 1

        # 检查数据库键
        for j in range(DEFAULT_KEY_NUMBERS):

            # 如果数据库中没有一个键带有过期时间，那么跳过这个数据库
            if redisDb.expires.size() == 0: break

            # 随机获取一个带有过期时间的键
            key_with_ttl = redisDb.expires.get_random_key()

            # 检查键是否过期，如果过期就删除它
            if is_expired(key_with_ttl):
                delete_key(key_with_ttl)

            # 已达到时间上限，停止处理
            if reach_time_limit(): return
```

#### 持久化、复制影响

Redis默认提供了RDB和AOF两种持久化方式，过期策略对它们处理方式也不尽同。

对于RDB持久化方式来说，生成RDB文件时，Redis默认会自动过滤过期键;而载入RDB文件时，又分两种情况:
以主服务器启动，Redis在载入过程中会过滤过期键，以从服务器启动，不论键是否过期，都会被载入到数据库。
不过，因为主从服务器在进行数据同步的时候，从服务器的数据库就会被清空，所以一般来讲，过期键对载入RDB文件的从服务器也不会造成影响。

对于AOF、复制来说，Redis处理方式基本一样，当删除一个过期键之后，显示追加一条DEL命令到AOF文件中，
同时也会显式地向所有从服务器发送一个DEL命令，告知从服务器删除这个过期键。从服务器只有在接到主服务器发来的DEL命令之后，才会删除过期键。


#### 参考

[Redis设计与实现](https://read.douban.com/ebook/7519526/?dct=Web&type=paid&dcc=7519526&dcm=douban&dcs=updates)

[Redis Document](https://redis.io/commands/expire)