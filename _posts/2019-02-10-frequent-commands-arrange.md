---
layout:     post
title:      "常用命令整理"
date:       2019-02-10 15:00:00 +0800
author:     "Sky丶Memory"
header-img: "img/2019-02-10-01-bg.jpg"
tags:
    - 工具


---

日常开发中，或多或少会使用一些工具来排查问题，本文整理下自己常用的命令供以后参考。

#### lsof

`lsof`是`list open files`的缩写，主要用于查看当前打开的文件。

查看某个进程下打开的文件：

`lsof -p <pid>`

查看某个文件被哪些进程打开：

`lsof -t  <filepath>`

`-i`是个比较常用的选项，其选项参数形式为：

`[46][protocol][@hostname|hostaddr][:service|port]`

- `46`：`IPV4`、`IPV6`
- `protocol`：`TCP`、`UDP`

比如查看`8000`端口被哪个进程占用:

`lsof -i:8000`

查看监听套接字：

`lsof -itcp -stcp:listen -nP`

#### netstat

`netstat`用于打印网络连接、路由表、连接的数据统计、伪装连接以及广播域成员。

查看当前所有的网络连接：

`netstat -a`

可通过`-t`、`-u`选项指明`tcp`、`udp`协议

查看监听套接字连接：

`netstat -nltp`



