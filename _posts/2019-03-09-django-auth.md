---
layout:     post
title:      "Django登录、认证机制"
date:       2019-03-09 17:00:00 +0800
author:     "Sky丶Memory"
header-img: "img/2019-03-09-01-bg.jpeg"
tags: Django Python



---

最近，新接手一个系统，用户模型依然继承自**AbstractBaseUser**，但表中明显有些字段是没有用的，所以打算研究研究在Django中如何能随心所欲的定义自己的**AUTH_USER_MODEL**，并复用其提供的**AuthenticationMiddleware**、**SessionMiddleware**

#### 解决方案

Django认证执行的代码流程如下所示：

```python
# django/contrib/auth/__init__.py
def load_backend(path):
    return import_string(path)()


def _get_backends(return_tuples=False):
    backends = []
    for backend_path in settings.AUTHENTICATION_BACKENDS:
        backend = load_backend(backend_path)
        backends.append((backend, backend_path) if return_tuples else backend)
    if not backends:
        raise ImproperlyConfigured(
            'No authentication backends have been defined. Does '
            'AUTHENTICATION_BACKENDS contain anything?'
        )
    return backends


def get_backends():
    return _get_backends(return_tuples=False)

def authenticate(request=None, **credentials):
    """
    If the given credentials are valid, return a User object.
    """
    for backend, backend_path in _get_backends(return_tuples=True):
        try:
            inspect.getcallargs(backend.authenticate, request, **credentials)
        except TypeError:
            # This backend doesn't accept these credentials as arguments. Try the next one.
            continue
        try:
            user = backend.authenticate(request, **credentials)
        except PermissionDenied:
            # This backend says to stop in our tracks - this user should not be allowed in at all.
            break
        if user is None:
            continue
        # Annotate the user object with the path of the backend.
        user.backend = backend_path
        return user

    # The credentials supplied are invalid to all backends, fire signal
    user_login_failed.send(sender=__name__, credentials=_clean_credentials(credentials), request=request)
```

默认情况下，**AUTHENTICATION_BACKENDS**值为['django.contrib.auth.backends.ModelBackend']，对于每一个backend类需提供authenticate接口实现基本认证功能

认证通过后，后端即可登录分配sessionid、记录sessionid对应的数据结构，默认情况下，Django中sessionid对应的数据结构为：

```javascript
{
    '_auth_user_id': '5',
    '_auth_user_backend': 'myauth.backends.ModelBackend',
    '_auth_user_hash': ''
}
```

刚看这部分源码时，比较有疑问的是—这里为什么需要记录**_auth_user_hash**，典型的一种情况：用户更改密码后，之前的sessionid都应失效，这也是记录**_auth_user_hash**的一个原因

后续用户带上sessionid请求后端时，后端通过AuthenticationMiddleware完成认证，认证代码逻辑如下：

```python
# django/contrib/auth/__init__.py
def get_user(request):
    """
    Return the user model instance associated with the given request session.
    If no user is retrieved, return an instance of `AnonymousUser`.
    """
    from .models import AnonymousUser
    user = None
    try:
        user_id = _get_user_session_key(request)
        backend_path = request.session[BACKEND_SESSION_KEY]
    except KeyError:
        pass
    else:
        if backend_path in settings.AUTHENTICATION_BACKENDS:
            backend = load_backend(backend_path)
            user = backend.get_user(user_id)
            # Verify the session
            if hasattr(user, 'get_session_auth_hash'):
                session_hash = request.session.get(HASH_SESSION_KEY)
                session_hash_verified = session_hash and constant_time_compare(
                    session_hash,
                    user.get_session_auth_hash()
                )
                if not session_hash_verified:
                    request.session.flush()
                    user = None

    return user or AnonymousUser()

```

结合刚才提到的backend类需要提供authenticate接口，这里还需提供get_user接口，故backend类总共需要提供两个接口：authenticate和get_user

看上去实现自定义backend类即可满足我们的需求，但是这里面有其他的坑，这里我只给出解决方案，细节层面的问题可自行深入了解

settings.py文件中加入如下配置：

```python
AUTH_USER_MODEL = 'myauth.User'
AUTHENTICATION_BACKENDS = ['myauth.backends.ModelBackend']
```

**注意，这里的配置需根据自己的项目目录自定义**

自定义用户模型、backend类，实现基本接口即可：

```python
from django.contrib.auth import get_user_model

UserModel = get_user_model()


class ModelBackend:
    def authenticate(self, request, phone, verify_code):
        pass

    def get_user(self, user_id):
        pass

class User(models.Model):
    phone = models.CharField(max_length=20, unique=True)
    address = models.CharField(max_length=50)
    age = models.IntegerField()

    REQUIRED_FIELDS = []
    USERNAME_FIELD = 'id'

    @property
    def is_anonymous(self):
        return False

    @property
    def is_authenticated(self):
        return True
 
 
```

需要说明一点，用户模型里面定义了**REQUIRED_FIELDS**、**USERNAME_FIELD**，这是由于auth包下的checks.py检查机制导致的，该模块定义了AUTH_USER_MODEL的一些约束，为避免应用启动不起来而添加的

