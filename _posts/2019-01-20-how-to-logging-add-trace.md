---
layout:     post
title:      "logging引入trace"
date:       2019-01-20 00:00:00 +0800
author:     "Sky丶Memory"
header-img: "img/2019-01-20-01-bg.jpeg"
tags:
    - logging
    - Python


---

#### 背景

在实际项目中，一个业务系统往往依赖多个微服务，各个微服务又或多或少的进行了水平扩展来避免单点问题，这样带来的一个问题就在于：问题的排查变得困难，不同服务之间的调用很难关联上。故而引出了全链路跟踪系统，旨在解决链路调用问题，比如美团MTrace、阿里EagleEye、Zipkin等。

本文并不打算介绍全链路跟踪系统的设计，只是提供一个问题的解决思路：如何在业务方无感知的情况下，让业务日志关联跟踪上下文

---

#### 思路

logging是Python标准库提供给开发者记录日志的模块，其处理流程图大概如下所示:

![](/img/2019-01-20-01-01.png)

Logger、Handler都有自己对应的日志级别、日志过滤器，这种设计的好处就在于：Logger可以做粗粒度的过滤、Handler做细粒度的过滤。

关于Filter是通过方法addFilter添加到当前Logger或Handler中，但我们可以定制化自己想要的Filter，只用实现其需要的接口即可

```python
class Filter(object):
    """
    Filter instances are used to perform arbitrary filtering of LogRecords.

    Loggers and Handlers can optionally use Filter instances to filter
    records as desired. The base filter class only allows events which are
    below a certain point in the logger hierarchy. For example, a filter
    initialized with "A.B" will allow events logged by loggers "A.B",
    "A.B.C", "A.B.C.D", "A.B.D" etc. but not "A.BB", "B.A.B" etc. If
    initialized with the empty string, all events are passed.
    """
    def __init__(self, name=''):
        """
        Initialize a filter.

        Initialize with the name of the logger which, together with its
        children, will have its events allowed through the filter. If no
        name is specified, allow every event.
        """
        self.name = name
        self.nlen = len(name)

    def filter(self, record):
        """
        Determine if the specified record is to be logged.

        Is the specified record to be logged? Returns 0 for no, nonzero for
        yes. If deemed appropriate, the record may be modified in-place.
        """
        if self.nlen == 0:
            return 1
        elif self.name == record.name:
            return 1
        elif record.name.find(self.name, 0, self.nlen) != 0:
            return 0
        return (record.name[self.nlen] == ".")
```

Filter对外提供的接口仅需要一个，也就是上述代码中的filter方法，所以我们可以通过Filter注入全链路跟踪信息

```python
import uuid
import logging
import threading
from collections import namedtuple


class TraceManager(object):
    _trace_ctx = threading.local()

    @classmethod
    def set(cls, **kwargs):
        ctx = cls._trace_ctx
        for k, v in kwargs.iteritems():
            if k.startswith('_'):
                raise KeyError("kwargs can't startswith _")
            setattr(ctx, k, v)

    @classmethod
    def unset(cls, *attrs):
        ctx = cls._trace_ctx
        for attr in attrs:
            if attr.startswith('_'):
                raise ValueError("attr can't startswith _")
            try:
                delattr(ctx, attr)
            except AttributeError:
                pass

    @classmethod
    def all(cls):
        ctx = cls._trace_ctx
        attrs = filter(lambda x: not x.startswith('_'), dir(ctx))
        dt = dict()
        for attr in attrs:
            dt[attr] = getattr(ctx, attr)

        return namedtuple('TraceCtx', attrs)(**dt)

    @classmethod
    def clear(cls):
        ctx = cls._trace_ctx
        attrs = filter(lambda x: not x.startswith('_'), dir(ctx))
        cls.unset(*attrs)


class TraceFilter(object):
    def filter(self, record):
        ctx = TraceManager.all()
        record.trace_id = getattr(ctx, 'trace_id', '-')
        return True


logger = logging.getLogger('test')
logger.setLevel(logging.DEBUG)

formatter = logging.Formatter('trace_id=%(trace_id)s level=%(levelname)s message=%(message)s')
handler = logging.StreamHandler()
handler.setLevel(logging.DEBUG)
handler.setFormatter(formatter)

logger.addFilter(TraceFilter())
logger.addHandler(handler)

TraceManager.set(trace_id=uuid.uuid1())

logger.info('Is ok?')
logger.debug("yeah, look fine.")

TraceManager.clear()

```

终端运行脚本文件，输出如下：

```sh
➜ python demo.py
trace_id=18149b9e-1c56-11e9-bbe2-f218987397c7 level=INFO message=Is ok?
trace_id=18149b9e-1c56-11e9-bbe2-f218987397c7 level=DEBUG message=yeah, look fine.
```

由此可见，这种方式是可行的，只是从设计的角度来说，并不雅观，因为Filter的功能只是做过滤，也就是单纯的读操作，这里已经变成了写操作。

这里再聊聊跟踪上下文的注入——TraceManager的set方法什么时候调用

一个系统内的流量来源，我理解的分三类：同步调用、异步调用、定时任务

- 同步调用：大多数服务之间的调用都属于应用层协议，比如常见的http，在接收到请求后、处理前注入当前请求的上下文即可
- 异步调用：异步调用往往会伴随一个id，类似Celery中task_id、MNS中msg_id，一种想法就是让异步调用的id和traceId保持一致
- 定时任务：id分配规则与traceId一直即可



