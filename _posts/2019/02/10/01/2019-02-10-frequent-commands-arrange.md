---
layout:     post
title:      "常用命令整理"
date:       2019-02-10 15:00:00 +0800
author:     "Sky丶Memory"
header-img: "img/2019/02/10/01/bg.jpg"
tags: Tools


---

日常开发中，或多或少会使用一些工具来排查问题，本文整理下自己常用的命令供以后参考。

---

#### ps

*ps*命令是*Process Status*缩写，主要用于查看当前运行的进程信息。

输出选项:

- *u*：包含*cpu*、*memory*使用率
- *-f*：包含父子进程*ID*
- *—forest*：*ascii*方式显示进程树

过滤选项：

- *-u*：显示指定用户下的进程
- *-C*：显示匹配的命令名称进程
- *-p*：显示指定进程信息
- *—ppid*：显示父进程为某*ID*的进程
- *-L*：显示进程下对应的线程信息，*lwp*指线程*id*，*nlwp*指线程数

排序选项：

- *—sort*：指定排序规则，如*—sort=-pcpu,+pmem*

---

#### strace

*strace*用于跟踪进程产生的系统调用和接收的信号。

常用选项:

- *-p*：指定跟踪进程
- *-f*：跟踪当前进程及由当前进程衍生的子进程、线程信息
- *-o*：指定输出文件
- *-e*：指定跟踪修饰符，用于过滤
- *-c*：统计某段时间内系统调用
- *-T*：显示系统调用耗时
- *-r*：显示系统调用相对时间
- *-s*:终端每行系统调用默认显示32个字符，会导致一些信息截断，通过该选项可设置
- *-y*：显示文件描述符对应的路径
- *-yy*：显示套接字文件描述信息

---

#### top

*top*命令用于实时查看进程信息，在交互式窗口下*top*支持不少的选项。这里说的选项指的是在交互式环境下输入对应的字母。

排序选项:

- *P*：*CPU*使用率排序
- *M*：内存使用率排序
- *T*：*CPU*占用时间排序

交互选项：

- *z*：单色与多色之间切换
- *b*：粗体展示
- *x*：高亮排序列
- *y*：高亮运行任务

过滤选项：

- *u*：指定用户进程
- *o*或*O*：指定过滤条件
- *L*：搜索指定文本

其他常用选项：

- *L*：显示对应的线程
- *k*：发送*SIGTERM*信号给指定进程
- *d*：调整刷新频率
- *c*：显示完整命令

---

#### lsof

*lsof*，是*list open files*简称，主要用于查看当前打开的文件。

常用选项：

- *-p*：查看某进程下打开的文件
- *-t*：查看某文件被哪些进程打开
- *-s p:s*：可用于指定查看协议及状态，比如查看监听套接字*lsof -itcp -stcp:listen -nP*
- `-i [46][protocol][@hostname|hostaddr][:service|port]`:用于查看网络套接字，如本机*8888*端口占用情况 *lsof -i:8888*

---

#### netstat

netstat，用于列出系统上所有的网络套接字连接情况，常用选项：

- *-a*：列出监听、非监听套接字
- *-t*或*-u*：列出*TCP*或*UDP*相关协议
- *-n*：禁用域名解析
- *-l*：列出监听套接字
- *-p*：显示进程相关信息
- *-s*：打印网络统计数据

---

#### ss

ss(socket statistics)，用于查看套接字信息，选项格式：

```shell
$ ss [ OPTIONS ] [ STATE-FILTER ] [ ADDRESS-FILTER ]
```

查看*established*状态*tcp*连接：

```shell
$ ss -t4 state established
Recv-Q Send-Q         Local Address:Port             Peer Address:Port   
0      0                192.168.1.2:54436          165.193.246.23:https   
0      0                192.168.1.2:43386          173.194.72.125:xmpp-client 
0      0                192.168.1.2:38355           199.59.150.46:https   
0      0                192.168.1.2:56198          108.160.162.37:http
```

查看目的端为*http*或*https*连接:

```shell
$ ss -nt dst :443 or dst :80
```

指定端口过滤:

```shell
$ ss -at 'dport = :ssh or sport = :ssh'
State      Recv-Q Send-Q    Local Address:Port        Peer Address:Port   
LISTEN     0      128                   *:ssh                    *:*       
LISTEN     0      128                  :::ssh                   :::*
```

