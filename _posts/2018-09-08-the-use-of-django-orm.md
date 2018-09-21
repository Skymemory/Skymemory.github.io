---
layout:     post
title:      "Django ORM常见使用"
date:       2018-09-08 10:30:00 +0800
author:     "Sky丶Memory"
header-img: "img/2018-09-08-02-bg.jpeg"
tags:
    - Python
    - Django
    - ORM
---

## 读写分离

ORM对Model操作默认都在default数据库中，在实际开发中，项目配置的数据库往往不止一个，这种情况下，就需要ORM提供路由功能:针对不同的Model操作，路由到不同数据库中。ORM默认提供了两种方式支持路由:

- Router: 通过指定路由规则，后续针对Model的操作ORM会根据配置规则自动路由到对应的Database
- Using: 对Model进行操作时，手工明确指定对应的Database

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
- django.db.Model类中定义save和delete方法，支持using参数明确指定使用的Database
- QuerySet类中实现using方法来明确指定使用的Database

当手工指定、路由规则都使用时，手工指定的优先级更高.一般情况下，常见的是使用路由规则，毕竟在写业务逻辑时，较少去关心这个查询走哪里啊什么之类的.
