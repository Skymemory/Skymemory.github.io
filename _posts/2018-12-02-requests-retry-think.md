---
layout:     post
title:      "关于requests库重试的一点感想"
date:       2018-12-02 02:30:00 +0800
author:     "Sky丶Memory"
header-img: "img/2018-12-02-01-bg.jpeg"
tags:
    - requests
    - Python
---

#### 背景

在外部服务调用过程中，由于网络的不可靠、服务限流等因素，导致调用方调用失败，在此类场景下，调用方有可能会对一些幂等性接口进行重试。

在Python中，用的比较多的http库当属requests，其简洁的接口设计、简便的使用方式吸引了众多开发者的青睐，所以这篇文章主要聊聊requests库的重试功能

#### 想法

requests自身并没有实现网络层，其网络层的依赖库为urllib3，所以requests完全可以看成是构建于urllib3之上的抽象程度更高的API。

清楚了这一点，就很容易从urllib3入手，幸运的是，urllib3提供重试功能，具体的[定义原型](https://urllib3.readthedocs.io/en/latest/reference/urllib3.util.html#module-urllib3.util.retry)如下:

```python
# urllib3/util/retry.py
class Retry(total=10, connect=None, read=None, redirect=None, status=None,
            method_whitelist=frozenset(['HEAD', 'TRACE', 'GET', 'PUT', 'OPTIONS', 'DELETE']), 
            status_forcelist=None,
            backoff_factor=0, raise_on_redirect=True, raise_on_status=True, history=None,
            respect_retry_after_header=True,
            remove_headers_on_redirect=frozenset(['Authorization'])):
    pass
```

重要参数说明:

- total: 重试总次数
- connect: 连接错误重试次数
- read: 读数据错误重试次数
- method_whitelist: 指定哪些http方法可以重试，一般来说，重试的http方法是幂等的
- status_forcelist: 指定哪些状态码可以重试
- backoff_factor: 指数退避因子

了解了Retry的基本定义，再来看下怎么在requests中定制化想要的重试。

稍微翻过requests库源码的人大概都清楚，requests是通过HTTPAdapter来屏蔽上下层之间的耦合，这样做的显著好处就是，
下层的变更对上层无感知，只需保证对上层的输出不变即可。

所以，我们很容易想到一个解决办法:
```python
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry


def requests_retry_session(session=None, total=3,
                           status_forcelist=(500, 502, 504), backoff_factor=0.3, **kw):
    retry = Retry(
        total=total,
        status_forcelist=status_forcelist,
        backoff_factor=backoff_factor,
        **kw
    )
    session = session or requests.Session()
    adapter = HTTPAdapter(max_retries=retry)
    session.mount('https://', adapter)
    session.mount('http://', adapter)
    return session
```

#### 验证

通过一个简单的测试例子验证下想法:
```python
def test():
    import time
    begin = time.time()
    try:
        r = requests_retry_session().get('http://127.0.0.1:8888/api/health')
    except Exception as e:
        print e
    else:
        print "Success. content: {}".format(r.content)
    finally:
        end = time.time()
        print 'elapsed time: {}'.format(end - begin)


if __name__ == '__main__':
    test()
```
本地没有启动对应端口为8888的server，所以得到了如下结果:
```shell
HTTPConnectionPool(host='localhost', port=8888): Max retries exceeded with url: /info (Caused by NewConnectionError('<urllib3.connection.HTTPConnection object at 0x106be7890>: Failed to establish a new connection: [Errno 61] Connection refused',))

elapsed time: 1.81431007385
```

1.8 = 0 + 0.6 + 1.2，具体的计算公式为:

> A backoff factor to apply between attempts after the second try (most errors are resolved immediately by a second try without a delay). urllib3 will sleep for:

> {backoff factor} * (2 ^ ({number of total retries} - 1))

> seconds. If the backoff_factor is 0.1, then sleep() will sleep for [0.0s, 0.2s, 0.4s, …] between retries. It will never be longer than Retry.BACKOFF_MAX.

> By default, backoff is disabled (set to 0).

另外，一般在调用过程中，会明确的指定timeout标识请求的超时时间，那再看看配合timeout参数是否符合预期。
```python
def test():
    import time
    begin = time.time()
    try:
        r = requests_retry_session().get('http://127.0.0.1:8888/delay/10', timeout=5)
    except Exception as e:
        print e
    else:
        print "Success. content: {}".format(r.content)
    finally:
        end = time.time()
        print 'elapsed time: {}'.format(end - begin)


if __name__ == '__main__':
    test()
```
本地启动端口为8888的server，在/delay/10视图中sleep 10s，输出如下：
```shell
HTTPConnectionPool(host='127.0.0.1', port=8888): Max retries exceeded with url: /delay/10 (Caused by ReadTimeoutError("HTTPConnectionPool(host='127.0.0.1', port=8888): Read timed out. (read timeout=5)",))

elapsed time: 21.8261728287
```

21.8 = 5 + 0 + 5 + 0.6 + 5 + 1.2 + 5

经过上面的测试，验证了之前的想法是符合我们预期的

#### 番外篇

上面只是提供了个人的一点思路，最终怎么定制化Retry配置，还看具体的场景。

另外，想说的一个问题，在一些中小型公司，由于一些技术原因，微服务内部之间的调用没有抽象出一个RPC层，还是直接在业务层
使用http协议，更糟糕的是，同一个服务不同的人封装http client的方式又不尽相同，导致了client的各种混乱，这里提供一种简易的封装方式
来规避类似的问题，具体实现:

```python
from functools import partial
from urlparse import urljoin

import requests
from urllib3.util.retry import Retry
from requests.adapters import HTTPAdapter


def requests_retry_session(session=None, total=3,
                           status_forcelist=(500, 502, 504), backoff_factor=0.3, **kw):
    retry = Retry(
        total=total,
        status_forcelist=status_forcelist,
        backoff_factor=backoff_factor,
        **kw
    )
    session = session or requests.Session()
    adapter = HTTPAdapter(max_retries=retry)
    session.mount('https://', adapter)
    session.mount('http://', adapter)
    return session


class RPCException(Exception):
    def __init__(self, code=0, message='success', data=None):
        self.code = code
        self.message = message
        self.data = data


class RPCClient(object):
    http_method_names = frozenset(['get', 'post', 'put', 'patch', 'delete', 'head', 'options', 'trace'])
    
    def __init__(self, host='127.0.0.1', headers=None, session=None):
        self.host = host
        self.headers = {} if headers is None else headers
        self._session = session or requests.Session()

    def __getitem__(self, item):
        method, uri = item.split('/', 1)
        if method not in self.http_method_names:
            raise ValueError('prefix must be http support method.')
        return partial(self._send, method, uri)

    def _send(self, method, uri, **kwargs):
        if self.headers:
            kwargs.setdefault('headers', {}).update(self.headers)

        url = urljoin(self.host, uri)
        r = self._session.request(method, url, **kwargs)

        __profile = kwargs.pop('__profile', False)
        if __profile:
            print 'path:{} elapsed time: {}s'.format(uri, r.elapsed.total_seconds())

        r.raise_for_status()
        data = r.json()
        code, message, data = data['code'], data['msg'], data['data']
        if code == 0:
            return data
        raise RPCException(code=code, message=message, data=data)


if __name__ == '__main__':
    rpc = RPCClient(host='http://127.0.0.1/crm/hydra/')
    d = rpc['get/coupons/'](params={'coupon_type': '1'}, __profile=True)
    print d

```

