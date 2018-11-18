---
layout:     post
title:      "Celery基本使用"
date:       2018-11-18 12:30:00 +0800
author:     "Sky丶Memory"
header-img: "img/2018-11-18-01-bg.jpg"
tags:
    - 分布式任务队列
    - Python
---

Celery是一个基于Python开发的简单、灵活且可靠的分布式任务队列系统，主要关注于实时任务的处理，同时也提供对定时、周期性任务的支持，其基本架构图如图所示：

![](/img/2018-11-18-01-01.jpeg)

重要组件说明:

1. Producer: 任务生产者，通过调用Celery提供的API而产生任务并交给任务队列处理的都是任务生产者
2. Broker：消息中间件，接受任务生产者发送的消息
3. Consumer: 任务消费者，通过从broker中获取消息并消费
4. Celery Beat: 任务调度器，Beat进程会根据配置文件的内容，周期性地将需要执行的任务发送给broker
5. Result Store: 保存Consumer执行完任务的结果，以供查询

---

#### Broker

Celery目前支持的Broker有四种:*RabbitMQ*、*Redis*、*Amazon SQS*、*Zookeeper*，前三个都可用于生产环境中，*Zookeeper*还处在试验阶段。

Celery官方推荐的是RabbitMQ，Celery的作者Ask Solem Hoel最初在VMware就是为RabbitMQ工作的，Celery最初的设计就是基于RabbitMQ，
所以使用RabbitMQ会非常稳定，成功案例很多。如果使用Redis，则需要能接受发生突然断电之类的问题造成Redis突然终止后的数据丢失等后果。

为了简单起见，本文例子采用的是Redis

---

#### 一个简单的例子

同一文件夹下创建tasks.py、celerycfg.py两个文件

```py
# file: tasks.py

from celery import Celery


app = Celery('tasks')

app.config_from_object('celerycfg')


@app.task
def add(x, y):
    return x + y

@app.task
def feeds():
    print "It's feeds task."

```

Celery运行需要一个Celery类的实例，也就是上述代码中的app，在启动Celery时通过-A选项明确指定

```py
# file: celerycfg.py


# 使用Redis做Broker

broker_url = 'redis://localhost'

# 任务结果存放于Redis

result_backend = 'redis://localhost/1'

# 任务序列化方式为json

task_serializer = 'json'

# 消费者只处理任务序列化方式为json的消息

accept_content = ['json']

# 结果序列化方式同样为json

result_serializer = 'json'

# 结果过期时间

result_expires = 5 * 60 * 60

# 当指定队列不存在时，默认不创建

task_create_missing_queues = False
```

对于celerycfg.py文件中的配置，我们可以通过python -m celerycfg来检查是否语法正确，这里对一些配置选项进行简要说明:
- broker_url: 格式为transport://userid:password@hostname:port/virtual_host，除transport是必须的，其他部分可选配置
- result_backend: 支持多种方式，如rpc、database、redis等，具体可参见官方文档[result_backend](http://docs.celeryproject.org/en/latest/userguide/configuration.html#std:setting-result_expires)
- task_serializer: 默认为*json*，支持*pickle*、*yaml*、*msgpack*，也可自定义序列化方法
- accept_content: 默认为*json*，也支持其他类型
- result_expires: 结果过期时间，由celery.backend_cleanup任务定时清理过期的结果
- task_create_missing_queues: 建议此选项设置为False，Celery默认会创建没有定义在task_queues中的队列

关于更多配置选项的说明，可参考Celery官方文档[Configuration and defaults](http://docs.celeryproject.org/en/latest/userguide/configuration.html)

启动消费者进程:

`celery worker -A tasks -l DEBUG`

上述启动过程中终端会显示一些有帮助的信息，比如消息中间件和存储结果的地址、并发数量、任务列表等，可通过如上信息判断配置是否符合预期。

通过IPython调用add函数来验证下是否符合我们逾期:

```py
In [1]: from tasks import add

In [2]: r = add.delay(10, 20)

In [3]: r.result
Out[3]: 30

In [4]: r.successful()
Out[4]: True

In [5]: r.status
Out[5]: u'SUCCESS'

In [6]: r
Out[6]: <AsyncResult: 72dbaff2-11ee-4af0-afa0-f1d3200d3a19>

In [7]: r.task_id
Out[7]: '72dbaff2-11ee-4af0-afa0-f1d3200d3a19'
```
从对应的终端中，可以观察到消费者接受并消费了该任务:
```
[2018-11-18 15:41:53,299: INFO/MainProcess] Received task: tasks.add[72dbaff2-11ee-4af0-afa0-f1d3200d3a19]
[2018-11-18 15:41:53,299: DEBUG/MainProcess] TaskPool: Apply <function _fast_trace_task at 0x1045d2578> (args:(u'tasks.add', u'72dbaff2-11ee-4af0-afa0-f1d3200d3a19', {u'origin': u'gen64307@MemorydeMacBook-Pro.local', u'lang': u'py', u'task': u'tasks.add', u'group': None, u'root_id': u'72dbaff2-11ee-4af0-afa0-f1d3200d3a19', u'delivery_info': {u'priority': 0, u'redelivered': None, u'routing_key': u'celery', u'exchange': u''}, u'expires': None, u'correlation_id': u'72dbaff2-11ee-4af0-afa0-f1d3200d3a19', u'retries': 0, u'timelimit': [None, None], u'argsrepr': u'(10, 20)', u'eta': None, u'parent_id': None, u'reply_to': u'f728a365-0f4a-314e-9100-bcf74e86896f', u'shadow': None, u'id': u'72dbaff2-11ee-4af0-afa0-f1d3200d3a19', u'kwargsrepr': u'{}'}, '[[10, 20], {}, {"chord": null, "callbacks": null, "errbacks": null, "chain": null}]', u'application/json', u'utf-8') kwargs:{})
[2018-11-18 15:41:53,300: DEBUG/MainProcess] Task accepted: tasks.add[72dbaff2-11ee-4af0-afa0-f1d3200d3a19] pid:64289
[2018-11-18 15:41:53,312: INFO/ForkPoolWorker-12] Task tasks.add[72dbaff2-11ee-4af0-afa0-f1d3200d3a19] succeeded in 0.0114071560092s: 30
```
另外，有时排查问题时，我们只知道task_id，需要获取之前任务执行的结果怎么办呢？Celery提供了根据task_id来获取任务执行结果的方式:
```py
In [1]: from tasks import add

In [2]: add.AsyncResult('72dbaff2-11ee-4af0-afa0-f1d3200d3a19').get()
Out[2]: 30

In [3]: from celery.result import AsyncResult

In [4]: AsyncResult('72dbaff2-11ee-4af0-afa0-f1d3200d3a19').get()
Out[4]: 30
```

---

#### 指定队列
Celery默认会将任务存放在名为celery的队列，我们也可以创建应用程序自身需要的队列并指定路由规则来实现多队列的分工。

```py
# file: celerycfg.py

from kombu import Queue

# 使用Redis做Broker

broker_url = 'redis://localhost'

# 任务结果存放于Redis

result_backend = 'redis://localhost/1'

# 任务序列化方式为json

task_serializer = 'json'

# 消费者只处理任务序列化方式为json的消息

accept_content = ['json']

# 结果序列化方式同样为json

result_serializer = 'json'

# 结果过期时间

result_expires = 5 * 60 * 60

# 当指定队列不存在时，默认不创建

task_create_missing_queues = False

# 更改默认队列名

task_default_queue = 'default'

# 定义任务队列

task_queues = (
    Queue('default'),
    Queue('feeds')
)

# 指定路由规则

task_routes = {
    'tasks.feeds': 'feeds'
}
```
通过-Q参数指定消费者进程从指定队列中获取任务:

`celery worker -A tasks -Q feeds -l DEBUG`

上述worker只会执行feeds队列中的任务，通过这种方式，我们可以将一些优先级比较高的任务单独存放在一个队列中，由指定的worker进行消费，
其他相对不那么重要的任务存放在另外的队列中来避免任务处理延迟、任务堆积等问题

另外，**强烈建议更改默认队列名**，有些公司测试或pre环境为了图方便，不少服务部署到同一台机器上，在这种情况下，如果别人的broker地址和你的一样，
会导致任务丢失

---

#### Celery Beat

Celery Beat主要用于管理定时、周期性任务，任务由beat_schedule指定，在celerycfg.py中添加如下配置:
```py
from datetime import timedelta
beat_schedule = {
    'add': {
        'task': 'tasks.add',
        'schedule': timedelta(seconds=20),
        'args': (10, 10)
    }
}
```

beat_schedule配置项指定tasks.add任务每隔20S执行一次，执行时的参数为10和10

启动beat进程和对应的worker:

`celery beat -A tasks -l DEBUG`

`celery worker -A tasks -l DEBUG -Q default`

从终端中，我们可以看到对应任务的执行情况

---

#### 任务绑定、重试

有时在执行任务时，可能会遇到网络不佳等意外情况，此时需要任务进行一定次数的重试，Celery同样提供了任务重试功能，在tasks.py中添加如下任务:
```py
@app.task(bind=True)
def div(self, x, y):
    try:
        retval = x / y
    except ZeroDivisionError as e:
        raise self.retry(exc=e, countdown=10, max_retries=3)

    return retval
```
然后在IPython中进行异常参数的调用:

`In : r = div.delay(2, 0)`

在终端中，我们会发现对应的任务每隔10S就会进行重试，一共重试3次，然后抛出异常。

可能有的人会存在疑问，自己在div中控制重试逻辑不也ok么？类似采用[Exponential Backoff](https://en.wikipedia.org/wiki/Exponential_backoff)的思想，嗯，
看上去好像没什么差异，将上述代码中的worker节点启动2个，然后重新执行重试，你会发现两者之间的差异。

---

#### 未完待续

上面只是简单介绍了Celery使用，还有些更进一步的知识需要了解，比如:
- 消费、确认机制
- 框架集成
- AMQP协议

#### 参考链接

[使用Celery](https://zhuanlan.zhihu.com/p/22304455)

[异步任务神器](http://funhacks.net/2016/12/13/celery/)
