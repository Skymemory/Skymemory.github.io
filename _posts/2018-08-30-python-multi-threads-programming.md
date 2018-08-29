---
layout:     post
title:      "Python多线程编程"
date:       2018-08-30 00:30:00 +0800
author:     "Sky丶Memory"
header-img: "img/2018-08-30-01-bg.webp"
catalog: true
tags:
    - 多线程
    - Python
---

&emsp;&emsp; Python为多线程编程提供了两个模块:***threading***和***_thread***，_thread模块提供了多线程底

层支持,以低级原始的方式来处理和控制线程；threading模块基于_thread进行了高层次的封装，将线

程操作对象化，并提供了更丰富的功能。在实际开发应用中，推荐优先使用threading(不排除特殊情

况下需要用到_thread)进行多线程编程.本文以Python 3.5版本，讲述Python标准库提供的多线程编程

基本功能。

### 创建线程

&emsp;&emsp; 在Python中，threading.Thread对象封装了常规的线程控制方法，其提供了两种方式用于创建

自定义线程:初始化threading.Thread时明确指定可调用对象和在threading.Thread子类中重载run方

法，具体的使用方式可参见标准文档说明，这里提供两个简单使用的例子：

``` python
import time
import threading


def worker():
    t = threading.current_thread()
    print(t.name, "is running in function")

    time.sleep(10)


class MyThread(threading.Thread):
    def run(self):
        print(self.name, "is running in class")

        time.sleep(10)


if __name__ == "__main__":
    # 通过target指定线程运行的可调用对象，可调用对象的参数可通过args和kwargs指定
    # 具体可查看标准文档
    t1 = threading.Thread(target=worker, name="Thread-function")
    t1.start()
    
    # 子类重载run方法
    t2 = MyThread(name="Thread-class")
    t2.start()

    time.sleep(5)
    print("Currently active threads:")

    for thread in threading.enumerate():
        print(thread.name)
```

&emsp;&emsp; 关于Python中线程的使用上，有3点需要额外注意下：

- 创建线程会占用线程栈空间，而GIL的存在导致Python无法充分利用多核并发的优势，故线程数并非越多越好
- Thread对象提供了join方法控制调用线程和被跟踪线程之间的交互
- Thread对象的*daemon*属性用于指明主线程退出时，是否可以不用等待该线程的终止，默认情况下，*daemon*属性继承自当前线程，可自行更改，主线程以非守护方式启动(*daemon*值为False).关于为什么要区分*daemon* 与非*daemon*，stackoverflow上有个很好的[解释](https://stackoverflow.com/questions/190010/daemon-threads-explanation)

### 线程同步

#### Lock

&emsp;&emsp; 在操作系统中，又称互斥锁，主要用于协调线程间对共享资源的同步访问。在Python中，获取

锁时可指定基于阻塞或非阻塞方式，具体通过acquire方法的blocking参数指定。基于阻塞式的方式很

容易理解，而非阻塞式的方式从一定程度上来说已经失去了部分实时性，那它的应用场景具体体现在

什么地方？一个简单的例子，在实际开发中，经常会将一些热点数据存放于缓存中，这里假设缓存存

在于进程内，而一般缓存都具有时效性，在高并发情况下，缓存失效时，会导致提供缓存的数据源负

载偏高。在这种情况下，我们并不希望所有的请求都转到数据源上，期望的是每种缓存都只有一个请

求到对应的数据源来负责缓存的重建，那么如何在并发的情况下分配一个请求到数据源呢？锁获取天

然支持并发竞争，通过利用非阻塞式的获取锁方式返回值，很容易实现此种类似的功能。


#### RLock

&emsp;&emsp; RLock，俗称可重入锁(Reentrant Lock)，主要用于解决互斥锁存在的一些缺陷：在互斥锁的使

用场景中，有时会遇到这样一类场景：在已获知互斥锁的情况下，调用其他方法，而在被调用的其他

方法中，也对同样的共享资源做了互斥访问，这就产生了一个闭环——锁的拥有者在等待锁的释放。

针对这种情况，业界提供了一种通用的解决方法——可重入锁。可重入锁需要额外的记录锁拥有者的

信息，使用方式上也和互斥锁有一定的差异，主要体现在:互斥锁的释放线程并不一定是获得该互斥

锁的线程，但可重入锁的释放线程必须是获得该可重入锁的线程


#### local

&emsp;&emsp; local，俗称线程局部变量，用于多线程中局部化的状态管理。当使用线程池时，需特别注意

线程局部变量的清理问题，避免线程归还到池中还带有旧有的状态

#### Condition、Semaphore、Event、Barrier

&emsp;&emsp; 关于这几个线程同步方式，暂时用的不多，简单知道怎么用，感兴趣的可以看下标准库的具体

实现，后续在补上。

#### Timer

&emsp;&emsp; Timer,俗称定时器，可在指定的时间间隔到达时，执行特定的逻辑，默认以多线程的方式执行。



