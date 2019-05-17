---
layout:     post
title:      "TCP学习笔记"
date:       2019-05-17 15:00:00 +0800
author:     "Sky丶Memory"
header-img: "img/2019-05-17-01-bg.jpg"
tags: Network

---

**TCP(Transmission Control Protocol)**，即传输控制协议，主要解决了数据在网络上的可靠传输问题。其数据流向大概为：应用层的数据会到TCP层的Segment中，然后TCP层的Segment会到IP层的Packet中，然后再到以太网Ethernet的Frame中，到达对端后，自底向上解析数据并交给应用方。

---

#### TCP、IP首部

TCP首部结构图：

![](/img/2019-05-17-01-01.png)

一些字段说明：

- 序号(Sequence Number)：用于解决包乱序问题
- 确认序号(Acknowledgement Number)：用于解决不丢包问题
- 窗口大小(Window Size)：用于解决流控问题
- 控制位(Control Flag)：包的类型，用于状态机流转
- 首部长度(Data Offset)：表示多少个4字节的首部长度，由于该值只占4个bit，所以首部最大长度为$4*15=60$字节

IP首部结构：

![](/img/2019-05-17-01-02.png)

一些字段说明：

- 生存时间(TTL)：控制Packet存活时间
- 标志位：用于控制Packet是否分片

---

#### TCP状态机

状态流转图:

![](/img/2019-05-17-01-03.png)

---

#### 三次握手

握手的目的在于建立连接及同步双方后续通信需要的信息，比如*mss*、*ISN*、*window size*等。

三次握手过程：

![](/img/2019-05-17-01-04.png)

三次握手中还有个很重要的概念——TCP连接队列，其在服务端的作用：

![](/img/2019-05-17-01-05.jpg)

Linux在2.2版本后用了两个队列来存放SYN_RCVD和EASTABLISHED状态的连接，其设置通过如下配置控制:

- syns queue：通过/proc/sys/net/ipv4/tcp_max_syn_backlog控制
- accept queue:取listen参数和/proc/sys/net/core/somaxconn两者之间的最小值

平时故障排查中，可以通过如下方式定位是否由连接队列引起的问题。

运行命令*netstat -s*：

![image-20190517163134722](/img/2019-05-17-01-06.png)

其中*overflowed*指*accept queue*队列满了，*sockets dropped*指*syns queue*队列满了。

通过*ss -nlt*命令可以查看具体某个应用全连接队列大小及当前队列信息:

![image-20190517164050409](/img/2019-05-17-01-07.png)

其中*Send-Q*表示全连接队列大小，*Recv-Q*表示当前等待进全连接队列或在全连接队列中的个数，如果*Recv-Q*大于*Send-Q*，则表示全连接队列已满，有部分连接无法处理。

关于全连接队列若满了，后续进队列连接逻辑控制由/proc/sys/net/ipv4/tcp_abort_on_overflow控制：

- 0：表示服务端直接扔掉客户端传过来的*ack*，虽然客户端已经状态已经是*ESTABLISHED*，但服务端连接并没有建立
- 1：发送客户端RST包

---

#### MTU、MSS

MTU(**Maximum Transmit Unit**)，最大传输单元，物理接口（数据链路层）提供给其上层最大一次传输数据的大小，当前应用最多的物理接口是以太网。

MSS(**Maximum Segment Size**)，最大TCP分段大小，不包含TCP首部和option，只包含TCP数据段。

这里有个问题，为什么我们需要MTU、MSS。

先来看看MTU，IP层的Packet到达数据链路层后转换为Frame，由网卡将每个Frame发出，那这里面就有两个值得关注的问题：不同Frame谁先谁后和每个Frame占用网卡的时间，这跟操作系统中的进程管理类似，不过有一点区别就在于Frame不具备拆分性。

至于谁先谁后的问题可看成简单队列问题，那每个Frame占用网卡时间怎么控制。

假设当前传输字节数为*T*，以太网头信息占14字节，尾部校验和FCS占了4字节，所以一个数据帧的传输效率为$\frac {T-18} T$，从式子中可以看出，*T*越大，传输效率越高，但是*T*越大发送每个Frame占用的时间也越多，所以以太网标准取*T*值为1518，其传输效率为$98.8\%$，这里的*T*值也就是*MTU*。

清楚了*MTU*后，在看下*IP*分片、重组。

数据包到达目的地期间可能需要经历多个中间设备，不同中间设备*MTU*值可能不同，这样就有可能面临数据包无法被传送的问题。针对这个问题，*IP*层引入了分片、重组技术，具体来讲就是将一个*Packet*进行分片，然后在接收端重组，这部分逻辑是由*IP*首部标志位控制。

*MSS*主要是*TCP*层用于分段依据，分段原因在于——若交由*IP*层分片，则有如下缺点：

- 某个分片丢失，导致分片前的整个数据包重新发送
- 分片、重组占用计算机开销也不低

所有交由上层自己控制也算是一种不错的选择，当然了，这部分是个人理解。

---

#### 可靠性

*TCP*保证可靠性是通过消费确认机制实现，这里面涉及两个概念：超时定义、滑动窗口

数据发送超时时间并不是硬编码，而是通过某种算法动态计算出来的。

滑动窗口概念也很好理解，发送端发送多个数据段，然后等待接收端回传每个段确认消息即可。

关于超时定义算法、滑动窗口里面涉及到的东西并不少，暂时不做深究。

---

#### 四次挥手

*TCP*是全双工通信，所以四次挥手也很好理解。

*TCP*状态图中进入*TIME_WAIT*后会等待$2MSL$，主要原因：

- 确保对方有足够的时间收到*ACK*，若没有收到，对端重发*FIN*，刚好$2MSL$
- 避免连接复用带来的问题

#### 后续

暂时需要整理的就这些了，后面有用到的在补充。

#### 参考

[TCP那些事儿(上)](https://coolshell.cn/articles/11564.html/comment-page-1#comments)

[TCP那些事儿(下)](https://coolshell.cn/articles/11609.html)

[关于TCP半连接队列和全连接队列]([http://jm.taobao.org/2017/05/25/525-1/](http://jm.taobao.org/2017/05/25/525-1/))

[How TCP backlog works in Linux]([http://veithen.io/2014/01/01/how-tcp-backlog-works-in-linux.html](http://veithen.io/2014/01/01/how-tcp-backlog-works-in-linux.html))