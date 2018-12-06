---
layout:     post
title:      "Django ORM基本篇"
date:       2018-09-08 10:30:00 +0800
author:     "Sky丶Memory"
header-img: "img/2018-09-08-02-bg.jpeg"
tags:
    - Python
    - Django
    - ORM
---

#### 常用字段

| ORM | Database(以Mysql为例)|
| :------: | :------: |
| SmallIntegerField | smallint |
| IntegerField | integer |
| PositiveIntegerField | integer UNSIGNED |
| PositiveSmallIntegerField | smallint UNSIGNED |
| TextField | longtext |
| BinaryField | longblob |
| CharField | varchar(%(max_length)s) |
| DateField | date |
| DateTimeField | datetime |
| UUIDField | char(32) |
| OneToOneField | integer |

Django同样对字段提供一些操作选项，这里顺便罗列一些常见选项:

| 字段选项 | 说明 |
| :------: | :------: |
| null | 默认false,表明字段不可为空 |
| choices | 可迭代对象,如:[(val1, desc1),(val2, desc2)],数据库存储对应的val值 |
| db_index | true或false,字段是否需要建索引 |
| default | 值或可调用对象 |
| unique | 唯一索引 |
| verbose_name | 说明文字 |
| max_length | 字符字段所占用的最大长度 |
| auto_now_add | 第一次创建对象时，ORM会自动设置该字段值 |
| auto_now | 调用save方法，ORM会自动赋值当前时间，其他情况如:QuerySet.update不会自动更新该值|

针对choices选项的字段，Django默认提供了get_filedname_display()方法将转换字段值和字段说明

Model Meta常见选项:
```py
from django.db import models

class Meta:
    abstract = True
    app_label = 'person'
    indexes = [
        models.Index(fields=['last_name', 'first_name']),
        models.Index(fields=['first_name'], name='first_name_idx'),
    ]
    unique_together = (('driver', 'restaurant'),)
```
这里简单说下个人看法，Django中可通过字段选项、Meta选项两种方式创建索引(前者不能建联合索引)，不过更建议通过Meta选项指定，好处在我看来有两个:
 
 - 索引定义与字段定义分离，语意明确
 - 便于了解是否存在冗余的索引，举个简单例子:last_name通过字段选项创建索引，Meta中创建联合索引(last_name, first_name)，此时last_name已经是冗余索引，但维护代码的人也不一定能知道

---

#### CRUD

#####  插入

Django支持单条、批量数据插入方式，这里简要说明下批量插入方式例子:
```py
>>> Entry.objects.bulk_create([
...     Entry(headline='This is a test'),
...     Entry(headline='This is only a test'),
... ])
```
批量插入函数原型为:
`bulk_create(objs, batch_size=None)`
batch_size用于指定批量大小，比如objs中存在1000条数据，batch_size为100，则需要10次网络请求(一次100条)才能插入整个数据

##### 更新

更新数据一般使用Model对象提供的save方法或QuerySet.update方法，save方法在使用上有个坑，从一个简单的看起:
```py
o = FaceTaskPerf.objects.get(id=2)

o.taken_nums += 1

o.save()
```
本意只想将id为2的任务taken_nums加1(这里只是为了举例说明问题，有更好的方法实现)，但是ORM最终转换的sql却非如此:
```sql
UPDATE `crm_facetaskperf` SET `username` = 'admin', `product_id` = 10, `audit_date` = '2018-12-05', `taken_nums` = 2, `done_nums` = 1, `same_nums` = 1, `unsame_nums` = 0, `created_time` = '2018-12-05 09:10:32.836373', `last_modified` = '2018-12-06 07:29:04.267796' WHERE `crm_facetaskperf`.`id` = 2
```
这就是save方法的默认行为，一般的使用原则:

***对需要更新的字段，save时通过update_fields明确指定***

比如刚才的例子我们可以通过如下方式达到预期:
```py
o = FaceTaskPerf.objects.get(id=2)

o.taken_nums += 1

o.save(update_fields=['taken_nums'])
```

##### 删除

删除数据一般通过Model对象提供的delete方法或者QuerySet.delete方法进行删除

##### 查询

F对象用于引用字段值，例如:
```py
from django.db.models import F
Credit.objects.filter(id=2).update(amount=F('amount') + 99)

# 执行对应SQL:

# UPDATE `api_credit` SET `amount`=(`api_credit`.`amount` + 99) WHERE `api_credit`.`id` = 2
```

Q对象用于构建复杂查询，比如OR查询等，其实现了__or__、\__and__、\__invert__魔方方法，分别对应操作符&、|、not，简单使用例子:
```py
Poll.objects.get(
    Q(question__startswith='Who'),
    Q(pub_date=date(2005, 5, 2)) | Q(pub_date=date(2005, 5, 6))
)
# 等价于:

SELECT * from polls WHERE question LIKE 'Who%'
    AND (pub_date = '2005-05-02' OR pub_date = '2005-05-06')
```
QuerySet对象提供的filter、get、exclude方法都支持Q对象作为其位置参数

Django ORM同样支持聚合函数，使用方法:
```py
In [56]: from django.db.models.aggregates import Sum, Avg, Count, Max, Min

In [57]: rs = Credit.objects.filter(status__in=[1,2]).aggregate(sum=Sum('amount'), cnt=Count('id'), max=Max('amount'))
    ...:

In [58]: rs
Out[58]: {'cnt': 1031, 'max': 300100, 'sum': 303690289}
```
aggregate方法使用上存在的一个小细节:
```py
In [60]: rs = Credit.objects.filter(status__in=[10, 20]).aggregate(sum=Sum('amount'))

In [61]: rs
Out[61]: {'sum': None}
```
QuerySet查询结果集为空，会导致sum值取值为None，需要特殊处理

---

#### 读写分离

ORM默认对Model操作都在DATABASES['default']中进行，实际开发中，项目配置的数据库往往不止一个，此时就需要ORM提供路由功能:

**针对不同的Model操作，路由到不同数据库中。**

ORM提供了两种方式支持路由:

- Router: 指定路由规则，后续针对Model的操作ORM会根据配置规则自动路由到对应的Database
- QuerySet.using:通过QuerySet提供的using方法，手工指定

这里先了解下Router的方式是怎么实现自动路由。考虑一个简单的例子，某个项目下存在auth应用，其配置了两个数据库master和slave，现在需要针对master和slave做读写分离，在ORM中需要怎么做呢？首先需要定义一个类，其实现如下接口:

```
class AuthRouter(object):
    """
    A router to control all database operations on models in the
    auth application.
    """
    def db_for_read(self, model, **hints):
        """
        Attempts to read auth models go to auth_db.
        """
        if model._meta.app_label == 'auth':
            return 'slave'
        return None

    def db_for_write(self, model, **hints):
        """
        Attempts to write auth models go to auth_db.
        """
        if model._meta.app_label == 'auth':
            return 'master'
        return None

    def allow_relation(self, obj1, obj2, **hints):
        """
        Allow relations if a model in the auth app is involved.
        """
        if obj1._meta.app_label == 'auth' or \
           obj2._meta.app_label == 'auth':
           return True
        return None

    def allow_migrate(self, db, app_label, model_name=None, **hints):
        """
        Make sure the auth app only appears in the 'auth_db'
        database.
        """
        return True

```

上述接口就是一个ORM Router类需要自定义的标准接口(也可以只实现你关心的接口)，通过AuthRouter我们指定了路由分配的规则，另外，我们还需要将AuthRouter指定的路由规则告知ORM，以便其进行调度，这通过在项目中的settings.py文件下配置
```
# settings.py
DATABASE_ROUTERS = ["path.to.AuthRouter"]
```
这样，我们就简单实现了读写分离。另外，关于ORM是如何调度我们指定的路由规则，其具体的实现对应Django源代码中的db.utils.ConnectionRouter类，感兴趣的可以了解下

另外，我们也可以通过手工指定Model操作的Database，具体的使用规则如下:
- Model对象提供的save和delete方法，支持using参数明确指定使用的Database
- QuerySet通过调用using方法来明确指定使用的Database