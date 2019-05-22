---
layout:     post
title:      "Celery基本篇"
date:       2018-11-18 12:30:00 +0800
author:     "Sky丶Memory"
header-img: "img/2018-11-18-01-bg.jpg"
tags:
    - Celery
    - Python
---

Celery是一个基于Python开发的分布式任务队列系统，支持实时性、周期性任务，各组件关系图：

![](/img/2018-11-18-01-01.jpeg)

---

#### Broker

Celery支持多种Broker，完整支持列表[Brokers](http://docs.celeryproject.org/en/latest/getting-started/brokers/)，官方推荐使用RabbitMQ作为Broker，为简单起见，本文使用Redis作为例子介绍。

---

#### 一个简单的例子

同一文件夹下创建tasks.py、celerycfg.py两个文件，文件内容分别为：

```python
# file: tasks.py
from celery import Celery

celery_app = Celery('tasks')
celery_app.config_from_object('celerycfg')


@celery_app.task
def add(x, y):
    return x + y

```

```python
# file: celerycfg.py
import pytz

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

# 指定时区，消息中的日期时间都会格式化为对应时区isoformat
timezone = pytz.timezone('Asia/Shanghai')

```

完整的配置选项说明 [Configuration and defaults](http://docs.celeryproject.org/en/latest/userguide/configuration.html)

启动消费者进程:

`celery worker -A tasks -l DEBUG`

![celery-startup](/img/2018-11-18-01-01.png)

启动过程中，终端会显示当前worker节点信息（如：broker地址、注册任务、监听队列等），可用于判断worker节点配置是否符合预期。

Celery提供两种方式触发异步任务，调用任务提供的方法：`delay`和`apply_async`。

通过`celery shell -A tasks`打开终端：

```python
In [1]: r = add.delay(2, 3)

In [2]: r.task_id
Out[2]: 'a3c4eb27-11c9-42e6-9f05-57e17af63f68'

In [3]: r.status
Out[3]: u'SUCCESS'

In [4]: r.result
Out[4]: 5

In [5]: r
Out[5]: <AsyncResult: a3c4eb27-11c9-42e6-9f05-57e17af63f68>
```

`delay`方法会返回一个`AsyncResult`对象，通过该对象，可获取到对应任务id、任务执行结果、任务状态等信息

`delay`方法只是对`apply_async`方法做了简单的封装，`apply_async`支持更多的参数，如可通过指定countdown、eta、expires关键参数明确任务执行时间或任务过期时间:

```python
In [1]: import pytz

In [2]: import datetime

In [3]: eta = datetime.timedelta(minutes=10) + app.conf.timezone.localize(datetime.datetime.now())

In [4]: add.apply_async(args=(89, 10), eta=eta)
Out[4]: <AsyncResult: 7b3aeb2d-c0b2-4363-a4e6-bc8b355881f8>

In [5]: add.apply_async(args=(5, 10), countdown=20)
Out[5]: <AsyncResult: 82eea5d1-4ae6-4b78-8762-465cd92ec91f>

In [6]: add.apply_async(args=(5, 10), expires=20)
Out[6]: <AsyncResult: 48b6cced-1abd-4ac0-b4ff-f3a10e47e378>
```

根据task_id获取任务的状态、结果等信息：

```python
In [13]: from celery.result import AsyncResult

In [14]: ar = AsyncResult('48b6cced-1abd-4ac0-b4ff-f3a10e47e378')

In [15]: ar.status
Out[15]: u'SUCCESS'

In [16]: ar.result
Out[16]: 15
```

---

#### 指定队列

Celery默认会将任务路由到队列名为celery的队列中，也可以自定义队列名及路由规则将任务路由到不同队列避免任务积压导致处理延迟问题。

`celerycfg.py`、`tasks.py`文件中分别增加如下内容：

```python
# celerycfg.py
from kombu import Queue

# 指定默认队列名
task_default_queue = 'canal'
task_queues = (
    Queue('canal.feed'),
    Queue('canal'),
    Queue('canal.notify'),
)
task_routes = {
    'tasks.add': 'canal',
    'tasks.notify': 'canal.notify',
    'tasks.feed': 'canal.feed',
}
```
```python
# tasks.py
@celery_app.task
def notify():
    print "notify task is executing"

@celery_app.task
def feed():
    print "feed task is executing"
```

启动消费者时，通过`-Q`选项明确指定监听队列：

`celery worker -A tasks -l DEBUG -Q canal.feed,canal.notify`

Celery会将任务模块名和任务名拼接`module_name.task_name`，根据`task_routes`获取对应的`routing_key`。

个人建议默认队列名不要用celery名称，在一些公司中，不免会遇到同一个测试环境中部署几个不同服务，如果自己负责的服务测试环境broker地址和别的服务一样，可能会导致任务丢失，建议默认队列名为服务名，剩余队列以`service_name.name`命名。

---

#### Celery Beat

`Celery Beat`用于管理周期性任务，周期性任务通过`beat_schedule`配置项指定，在`celerycfg.py`中添加如下配置:

```python
from celery.schedules import crontab

beat_schedule = {
    'add': {
        'task': 'tasks.add',
        'schedule': crontab(minute='*/2'),
        'args': (10, 10),
    }
}
```

启动beat进程：

`celery beat -A tasks -l DEBUG`

每隔两分钟，通过worker终端，可看到任务执行情况，具体`crontab`说明，参见[crontab](http://docs.celeryproject.org/en/latest/reference/celery.schedules.html#celery.schedules.crontab)

---

#### 任务重试、撤销

`worker`在执行任务时，可能会遇到网络不可达等需要任务重试的情况，Celery同样提供了任务重试机制，在`tasks.py`中添加如下内容：

```python
@celery_app.task(bind=True)
def div(self, x, y):
    try:
        r = x / y
    except ZeroDivisionError as e:
        raise self.retry(exc=e, max_retries=3, countdown=20)
    return r
```

通过`celery shell`调用`div(2,0)`，从对应的终端，会看到任务执行了重试逻辑。

异步带来的一个典型好处在于可以撤销之前的任务，这在`Celery`中也可以做到。

```python
In [5]: from celery.task.control import revoke

In [6]: r = add.apply_async(args=(1,2), countdown=100)

In [7]: r.revoke()

In [8]: r1 = add.apply_async(args=(10, 20), countdown=200)

In [9]: revoke(r1.task_id)
```



---

#### Flower

`Flower`是基于`web`的监控和管理`celery`的工具，提供了常规监控功能，如队列长度、`worker`状态及统计、任务详细信息等，具体使用说明可参见官方文档[Flower](https://flower-docs-cn.readthedocs.io/zh/latest/)。

一般通过如下命令启动即可：

`flower -A tasks —-max_tasks=5000 --port=800`

---

#### 未完待续

本文只是简单介绍了Celery使用，还有些更进一步的知识需要了解，比如:
- 消费、确认机制
- AMQP协议
