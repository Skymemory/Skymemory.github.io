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

