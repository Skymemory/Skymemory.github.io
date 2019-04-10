---
layout:     post
title:      "Python搭建简易HTTPServer"
date:       2019-04-10 23:00:00 +0800
author:     "Sky丶Memory"
header-img: "img/2019-04-10-01-bg.jpg"
tags: Python HTTP


---

有时，为了验证一些想法，比如keep-alive具体行为、协程调度器调度算法等，需要一个简易的***HTTPServer***帮助我们快速验证此类想法，这里简单罗列了几种方法供以后参考。

***Python***标准库提供了一个简单的命令来实现文件目录共享的***HTTPServer***，终端运行命令`python -m SimpleHTTPServer`，会得到如下信息：

```powershell
Serving HTTP on 0.0.0.0 port 8000 ...
```

在浏览器中访问`8000`端口，会看到运行命令的目录结构及文件，这在局域网中可快速共享文件。

另外，在`Python 3.x`中，`SimpleHTTPServer`移动到了`http.server`模块下，运行命令变为`python3 -m http.server`

如果需要自定义请求处理的逻辑，可通过下面的代码实现（`Python 2.x`）:

```python
import sys
from BaseHTTPServer import BaseHTTPRequestHandler, HTTPServer


class TestHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        content = "<h1>Hello,SkyMemory</h1>"

        self.send_response(200)

        self.send_header('Content-Type', 'text/html')
        self.end_headers()

        self.wfile.write(content)


def test(protocol='HTTP/1.0'):
    if sys.argv[1:]:
        port = int(sys.argv[1])
    else:
        port = 8000
    BaseHTTPRequestHandler.protocol_version = protocol
    httpd = HTTPServer(('', port), TestHandler)
    sa = httpd.socket.getsockname()
    print "Serving HTTP on", sa[0], "port", sa[1], "..."
    httpd.serve_forever()


if __name__ == '__main__':
    test()
```

标准库提供的`HTTPServer`通过`do_method`方式来定义请求处理逻辑。

另外，有时需要`HTTPServer`支持并发，针对这点，`Python 2.x`中暂时想到的办法是通过`uwsgi`，`Python 3.x`中有更好的方法，后面会提到。

```python
# test.py
def application(environ, start_response):
    headers = [('Content-Type', 'text/html')]
    start_response("200 OK", headers)
    return "<h1>Hello,SkyMemory</h1>"
```

运行命令`uwsgi --http :8000 -p 4 --wsgi-file test.py`即可得到一个简易、支持并发的`HTTPServer`，具体使用哪种并发模式就看个人所需。

在`Python 3.x中`，标准库提供了一个支持并发处理的`ThreadingHTTPServer`，具体使用方法:

```python
import json
import sys
from http.server import ThreadingHTTPServer, BaseHTTPRequestHandler


class TestHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        data = json.dumps({
            'name': 'SkyMemory',
            'address': 'Beijing',
        })

        self.send_response(200)

        self.send_header('Content-type', 'application/json')
        self.end_headers()

        self.wfile.write(data.encode())


def test(protocol='HTTP/1.0'):
    if sys.argv[1:]:
        port = sys.argv[1]
    else:
        port = 8000
    TestHandler.protocol_version = protocol
    httpd = ThreadingHTTPServer(('', port), TestHandler)
    sa = httpd.socket.getsockname()
    print("Serving HTTP on", sa[0], "port", sa[1], "...")
    httpd.serve_forever()


if __name__ == '__main__':
    test()

```



