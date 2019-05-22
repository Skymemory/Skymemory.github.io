---
layout:     post
title:      "Docker学习笔记"
date:       2019-04-03 17:00:00 +0800
author:     "Sky丶Memory"
header-img: "img/2019-04-03-01-bg.jpg"
tags: Note Tech

---

容器是应用层的抽象，它将代码和依赖关系打包在一起，多个容器可以在同一台机器上运行，并与其他容器共享操作系统内核，每个容器在用户空间中作为独立进程运行。

如果将容器抽象成一种概念，那么`Docker`便是这种概念的一种实现方式。

根据官方文档的介绍，`Docker`具备如下特点：

- **轻量**：在一台机器上运行的多个Docker容器，可以共享这台机器的操作系统内核，它们能够迅速启动，只需占用很少的计算和内存资源。
- **标准**：Docker容器基于开放式标准，能够在所有主流Linux版本、Microsoft Windows以及包括VM、裸机服务器和云在内的任何基础设施上运行。
- **安全**：Docker赋予应用的隔离性不仅限于彼此隔离，还独立于底层的基础设施。Docker默认提供最强的隔离，因此应用出现问题，也只是单个容器的问题，而不会波及到整台机器。

不过，对于普通开发者来说，更看重的是它提供的环境隔离功能。

---



#### 基本概念

`Docker`是基于**`C/S`**架构的程序，通过使用`API`方式管理容器，基本架构：

![](/img/2019-04-03-01-01.png)

**镜像**类似一个引导文件系统，容器运行离不开镜像。

**容器**更像是系统中某个进程，但与直接在宿主执行的进程不同，容器进程运行于属于自己的独立命名空间。

镜像与容器的关系，就像是程序设计中的类与实例一样。

`Registry`是`Docker`用来管理镜像的仓库，如著名的官方仓库[Docker Hub](<https://hub.docker.com/>)。

---



#### 基本使用

安装好`Docker`之后，便可以通过命令`docker`来操作对应的镜像、容器等。

拉取`ubuntu`镜像：

```powershell
➜  ~ docker pull ubuntu:latest
latest: Pulling from library/ubuntu
898c46f3b1a1: Pull complete
63366dfa0a50: Pull complete
041d4cd74a92: Pull complete
6e1bee0f8701: Pull complete
Digest: sha256:017eef0b616011647b269b5c65826e2e2ebddbe5d1f8c1e56b3599fb14fabec8
Status: Downloaded newer image for ubuntu:latest
```

`pull`子命令可用于拉取仓库或镜像，仓库下可以存在多个镜像，若需明确拉取某个仓库下的某个镜像，可通过增加`:[tag]`方式明确具体的镜像。

关于镜像操作常用命令：

- `docker search name`：搜索`Registry`下匹配`name`的仓库或镜像，`Docker Hub`提供网页搜索功能
- `docker rmi image_name`：删除image_name镜像
- `docker images`：列出所有镜像

有了镜像，接下来便是启动对应的容器，启动`ubuntu`容器：

```powershell
➜  ~ docker run -d --name ubuntu -it ubuntu:latest /bin/bash
b633e8c2adeca72822b8a90fff660b25276cf8df6b67949961063086509bebb2
```

执行完对应的命令后，如果执行成功，会得到如上所示的一个16进制串，该字符串唯一标识了该容器。

选项说明：

- `-d`：表示容器后台运行
- `--name`：指定容器名称，容器名称或容器ID都可唯一表示一个容器实例
- `-it`：保证容器中`STDIN`开启并分配一个伪`tty`

容器创建后，即可通过`docker ps -a`查看容器状态：

```powershell
➜  ~ docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
b633e8c2adec        ubuntu:latest       "/bin/bash"         10 minutes ago      Up 10 minutes                           ubuntu
```

我们也可以通过`docker top ubuntu`查看容器内的进程：

```powershell
➜  ~ docker top ubuntu
PID                 USER                TIME                COMMAND
16922               root                0:00                /bin/bash
```

容器运行起来后，可通过`docker exec`子命令进入容器内：

```powershell
➜  ~ docker exec -it ubuntu /bin/bash
root@b633e8c2adec:/#
```

此时的ubuntu容器是个精简版的ubuntu系统，不过，大多数情况下还是能正常安装需要软件：

```powershell
➜  ~ docker exec -it ubuntu /bin/bash
root@b633e8c2adec:/# apt-get update && apt-get install -y redis
Get:1 http://archive.ubuntu.com/ubuntu bionic InRelease [242 kB]
Get:2 http://security.ubuntu.com/ubuntu bionic-security InRelease [88.7 kB]
Get:3 http://security.ubuntu.com/ubuntu bionic-security/restricted amd64 Packages [5436 B]
Get:4 http://archive.ubuntu.com/ubuntu bionic-updates InRelease [88.7 kB]
...........
invoke-rc.d: could not determine current runlevel
invoke-rc.d: policy-rc.d denied execution of start.
Setting up redis (5:4.0.9-1ubuntu0.1) ...
```

常见操作容器命令：

- `docker start container_name`：启动`container_name`容器
- `docker stop container_name`：停止`container_name`容器
- `docker rm container_name`：删除`container_name`容器
- `docker cp`：用于拷贝宿主机和容器之间的文件或文件夹
- `docker logs`：查看容器日志
- `docker inspect`：查看容器更详细的信息

`docker`提供的子命令基本上都可通过`docker subcommand -h`方式查看对应的帮助文档。

上面说了这么多，不过貌似看上去没啥用。。。。不急，现在才刚刚开始。

---



#### Dockerfile

容器的运行离不开镜像，上面例子中使用的都是别人制作好的镜像，如果有自定义镜像的需求呢，比想在某个容器内运行一个`Python`程序？

`Docker`提供了两种方式构建镜像：

- 使用`docker commit`命令
- 使用`docker build`和`Dockerfile`文件

运行`docker exec -it ubuntu /bin/bash`命令，安装`vim`、`python3-dev`:

```powershell
➜  ~ docker exec -it ubuntu /bin/bash
root@b633e8c2adec:/# apt-get install -y vim && apt-get install -y python3-dev
```

`/`目录下添加`test.py`文件:

```python
from http.server import HTTPServer, BaseHTTPRequestHandler
import json

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.send_header('Content-type', 'application/json')
        self.end_headers()
        self.wfile.write(json.dumps({
            'host': "localhost",
            "message": "This is a dockerfile test."
        }).encode())


if __name__ == '__main__':
    server = HTTPServer(('0.0.0.0', 8000), Handler)
    print("server starting.....")
    server.serve_forever()
```

退出容器，执行如下命令：

```powershell
➜  docker commit ubuntu skymemory/demo:0.1
sha256:b6396664ec770be583973ef0e09754be000ea6d14ff1dc76feedaacccb6514ce
➜  docker images
REPOSITORY                 TAG                 IMAGE ID            CREATED             SIZE
skymemory/demo             0.1                 b6396664ec77        9 seconds ago       358MB
```

此时，对应的`python`环境镜像就制作完成，启动对应容器及验证：

```powershell
➜  ~ docker run -d --name httpd -p 9999:8000 skymemory/demo:0.1 /bin/sh -c "python3 test.py"
ce31e6c3af48c4220e8619655eb09915bc4800b533d17d77526e0b14f646900f
➜  ~ docker top httpd
PID                 USER                TIME                COMMAND
19106               root                0:00                /bin/sh -c python3 test.py
19150               root                0:00                python3 test.py
➜  ~ http http://localhost:9999/home
HTTP/1.0 200 OK
Content-type: application/json
Date: Wed, 03 Apr 2019 11:21:11 GMT
Server: BaseHTTP/0.6 Python/3.6.7

{
    "host": "localhost",
    "message": "This is a dockerfile test."
}
```

现在再看看通过`Dockerfile`文件和`docker build`构建镜像过程。

创建目录`df`，目录下创建`Dockerfile`和`test.py`文件，`test.py`文件内容与之前一致，`Dockerfile`文件内容：

```dockerfile
FROM ubuntu:latest
RUN apt-get update
RUN apt-get install -y python3-dev
ADD test.py /
EXPOSE 8000
CMD ["python3", "test.py"]
```

在`df`目录下执行：

```powershell
➜ docker build -t skymemory/demo:0.2 .
Sending build context to Docker daemon  3.584kB
Step 1/6 : FROM ubuntu:latest
 ---> 94e814e2efa8
Step 2/6 : RUN apt-get update
.....
Successfully built 4b49663acdc2
Successfully tagged skymemory/demo:0.2
```

镜像成功构建后，执行如下命令：

```powershell
➜  docker run -d --name httpd-1 -p 8888:8000 skymemory/demo:0.2
e220ded3a673bd74539a2b2ca911b9926d9e0d8d4a49444d3b90a388d5c8a0dc
➜  docker top httpd-1
PID                 USER                TIME                COMMAND
20418               root                0:00                python3 test.py
➜  http http://localhost:8888/home
HTTP/1.0 200 OK
Content-type: application/json
Date: Wed, 03 Apr 2019 11:37:19 GMT
Server: BaseHTTP/0.6 Python/3.6.7

{
    "host": "localhost",
    "message": "This is a dockerfile test."
}
```

`Dockerfile`基于`DSL`语法的指令构建一个`Docker`镜像，常用的一些指令整理：

- `CMD`：指定容器启动时运行的命令
- `RUN`：指定镜像被构建时要运行的命令
- `WORKDIR`：设置工作目录，类似`cd`
- `ENV`：设置环境变量，格式为`ENV name value`
- `COPY`：构建时用于复制本地目录或文件到容器
- `EXPOSE`：指定暴露端口
- `VOLUME`：指定挂载卷（后续会说明）

---



#### Volume

在开发阶段代码频繁改动代码是很常见的事，如果通过上述构建镜像的方式运行项目代码，每更改一次就需要重新构建镜像，未免显得低效。

`Docker`中引入了`Volume`，又称"卷"，可用于将容器中的目录或文件挂载到外部文件系统，这样外部文件系统的更改就能反应到容器内文件系统上。

`Volume`在`Dockerfile`中通过`VOLUME`指令指定，如：

```powershell
VOLUME ["/data1", "/data2"]
```

也可在`docker run`命令中指定：

```powershell
docker run -d --name httpd-2 -v $PWD/test.py:/test.py -p 8899:8000 skymemory/demo:0.2
```

每当更新外部文件系统，根据`docker restart`命令重启对应的容器即可。

当然，`Volume`的功能并不仅限于此，感兴趣的可以自行了解。

---



#### link

现实场景中，一个服务往往需要多个容器，如`mongodb`容器、`redis`容器、`mysql`容器等。

由于容器每次重启对应的IP地址都会发生变化，这样就导致了某个容器重启后，依赖该容器的服务就需要更换调用IP地址，极大的不方便。

`Docker`提供了`link`功能来解决诸如此类问题，运行`docker run`命令时可通过—link选项指明连接容器及别名，如:

```powershell
docker run --link redis:cache --rm -it ubuntu /bin/bash
```

上述命令指定`link`容器名为`redis`，别名为`cache`，`Docker`内部会在当前容器`/etc/hosts`文件中添加`cache`到对应IP的转换，另外还会将`redis`容器中的环境变量导入本容器，根据这两点可以获知`link`容器信息。

---



#### docker-compose

还是之前的问题，一个服务往往涉及多个容器，如缓存、持久化存储、`oss`存储、搜索等，人工部署各个容器无疑加大人力成本。

我们需要一个工具来管理多个容器，这个工具需要提供服务编排的功能，这就是`docker-compose`。

通常情况下安装好`Docker`后，默认会安装`docker-compose`工具，`docker-compose`配置文件格式为`yml`:

```yaml
cache:
  image: redis

httpd:
  image: test:0.1
  links:
    - cache
  ports:
    - "8888:8000"
  volumes:
    - /Users/memorysky/demo:/tmp
```

定义好服务`yml`文件后，即可通过`docker-compose up -d`方式启动服务。

基本上平时自己会用到的内容大体就这些，其他的内容用到的时候在说吧。

