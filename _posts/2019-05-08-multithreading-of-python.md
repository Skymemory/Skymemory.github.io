---
layout:     post
title:      "Python之多线程编程"
date:       2019-05-08 15:00:00 +0800
author:     "Sky丶Memory"
header-img: "img/2019-05-08-01-bg.jpg"
tags: Python Concurrence Thread
usemathjax: true

---

现代CPU大多数都支持多核，充分利用多核计算的优势，是工程师必备的技能之一。

#### 多线程

Python中多线程编程有两种使用形式（C扩展方式不在本文讨论范围内）：

```python
import threading
import time

def calc_sum(n):
    ss = 0
    for i in range(n + 1):
        ss += i
    print("sun of n is {}.".format(ss))

class CountDownThread(threading.Thread):
    def __init__(self, n):
        super().__init__()
        self.n = n

    def run(self):
        while self.n > 0:
            print("Current n is {}.".format(self.n))
            self.n -= 1
            time.sleep(0.1)

if __name__ == '__main__':
    # way1
    t1 = threading.Thread(target=calc_sum, args=(10,))
    t1.start()

    # way2
    t2 = CountDownThread(10)
    t2.start()

```

*way2*方式有个缺点，任务与并发方式存在一定耦合。

Python中线程分两类：daemon thread和non-daemon thread，其中daemon thread官方的解释：

>
>
>A thread can be flagged as a “daemon thread”. The significance of this flag is that the entire Python program exits when only daemon threads are left. The initial value is inherited from the creating thread. The flag can be set through the [`daemon`](#threading.Thread.daemon) property or the *daemon* constructor argument.
>
>Daemon threads are abruptly stopped at shutdown. Their resources (such as open files, database transactions, etc.) may not be released properly. If you want your threads to stop gracefully, make them non-daemonic and use a suitable signalling mechanism such as an [`Event`](#threading.Event).
>
>There is a “main thread” object; this corresponds to the initial thread of control in the Python program. It is not a daemon thread.

大体意思就是说，daemon属性默认值继承自父线程（主线程是non-daemon thread），主线程退出时不必等待daemon thread。

至于为什么需要daemon thread，考察这样一类任务线程：*GC*、心跳包等，这类线程只在主线程运行时才有用，运行完毕后即可终止。若没有*daemon thread*，我们需要手动的跟踪这些线程并在主线程退出时，手动的关闭它们。通过设置成*daemon thread*，当主线程退出时，这类线程就被自动终止了（猜测是解释器托管了这些线程）。

*stackoverflow*上有个关于*daemon thread*的解释[*Daemon Threads Explanation*](https://stackoverflow.com/questions/190010/daemon-threads-explanation)

---

#### GIL

从一个简单例子看起：

```python
import threading
import time


def fib(n):
    if n <= 1:
        return n
    return fib(n - 1) + fib(n - 2)


def single_thread(times):
    r = []
    for i in range(times):
        tb = time.time()
        fib(30)
        fib(30)
        fib(30)
        te = time.time()
        r.append('{:.5f}'.format(te - tb))

    return r


def multi_thread(times):
    r = []
    for i in range(times):
        ts = []
        for _ in range(3):
            t = threading.Thread(target=fib, args=(30,))
            ts.append(t)

        tb = time.time()
        for t in ts:
            t.start()
        for t in ts:
            t.join()
        te = time.time()
        r.append('{:.5f}'.format(te - tb))
    return r


t1 = single_thread(20)
t2 = multi_thread(20)
print(t1)
print(t2)

```

在我$12$核电脑上运行结果为:

````shell
['0.84675', '0.86230', '0.87799', '0.84974', '0.85043', '0.85675', '0.82343', '0.82434', '0.83729', '0.84523', '0.83385', '0.83480', '0.82559', '0.82848', '0.83206', '0.82472', '0.82823', '0.82451', '0.83502', '0.82901']
['0.85296', '0.87298', '0.85680', '0.84538', '0.85485', '0.84925', '0.84522', '0.83931', '0.86896', '0.86092', '0.85658', '0.84691', '0.86612', '0.85414', '0.84543', '0.86851', '0.84650', '0.85566', '0.85589', '0.85259']

````

多次运行上述脚本并观察实验结果，会得出一个事实：单线程方式大多数时候都比多线程方式快。

是什么原因导致了这种现象？

首先，需要了解一个基本事实，Python存在多种解释器，常见的如CPython、Jython、PyPy等，用的比较多的解释器当属*CPython*，而*GIL*就是*CPython*中存在的问题，这也间接说明了*GIL*并不是*Python*语言的特性。

*CPython*中对*GIL*官方解释:

> *In CPython, the global interpreter lock, or GIL, is a mutex that prevents multiple native threads from executing Python bytecodes at once. This lock is necessary mainly because CPython’s memory management is not thread-safe. (However, since the GIL exists, other features have grown to depend on the guarantees that it enforces.)*

从上述解释中可以得知，是由于*CPython*内存管理是非线程安全导致了*GIL*存在，*GIL*限制了同一时刻只有一个***native thread***能执行字节码。

那*CPython*内存管理为什么是非线程安全，主要原因在于*Python*使用引用计数管理内存:

```python
>>> import sys
>>> a = []
>>> b = a
>>> sys.getrefcount(a)
3
```

在多线程环境下，引用计数方式会面临竞争条件，进而导致内存泄漏等其他问题，故而采用串化方式来规避了该问题。

那是不是用*CPython*解释器的*Python*程序多线程就没有用了。

当然不是，任务一般分为两种：*CPU*繁忙型和*IO*繁忙型，针对*IO*繁忙型，*Python*的多线程还是有一定优势，解释器在线程阻塞时会释放*GIL*以便其他线程执行。

另外，*CPython*中也是采用时间片轮转的方式进行线程切换，其间隔时间可以通过*sys.getswitchinterval*获知。

至此，也就能解释通为什么上述代码中多线程版本会比单线程版本慢了：申请锁、释放锁、线程切换存在一定的开销。

通过以上分析可以得知，在以*CPython*为执行解释器的*Python*程序中，多线程编程并不适合*CPU*繁忙型任务。

---

#### RLock

RLock，名为可重入锁，其应用场景：一个线程获取到锁$lock_1$后、未释放$lock_1$之前又调用了需要获取$lock_1$的函数。

遗憾的是，在工作中没有碰上过使用*RLock*的场景，这里简单和*Lock*做下对比:

- *Lock*对获取锁、释放锁线程没有要求，*RLock*要求释放锁线程必须是获得锁线程

更确切的说，*RLock*只是*Lock*更高层的抽象，内部通过记录*owning thread*和*recursion level*即可。

---

#### Condition Variable

条件变量，可看成是一种场景模型，拿生产者、消费者模型来说，生产者向队列中放入元素时先获取到*lock*后，需判断队列是否已满，若已满，则挂起自身。这里面的是否已满就是个条件，条件变量处理的场景几乎都类似于此，后续提到的*Semaphore*、*Event*内部都是基于条件变量实现的。

条件变量处理场景模型:

```python
# Consume one item
with cv:
    while not an_item_is_available():
        cv.wait()
    get_an_available_item()

# Produce one item
with cv:
    make_an_item_available()
    cv.notify()
```

---

#### Semaphore

semaphore，名为信号量，可用于线程同步、资源访问限制，比如限制同一时刻至多有$3$个线程访问打印机：

```python
import time
import threading

limit_sema = threading.Semaphore(3)


def printer(text):
    current_thread = threading.current_thread()
    with limit_sema:
        # simulate time-consuming operation
        time.sleep(2)
        print('{} thread print content:{}'.format(current_thread.name, text))


ts = []
for i in range(10):
    t = threading.Thread(target=printer, args=(str(i), ))
    ts.append(t)
    t.start()

for t in ts:
    t.join()

```

运行输出:

```shell
Thread-1 thread print content:0
Thread-2 thread print content:1
Thread-3 thread print content:2

Thread-4 thread print content:3
Thread-5 thread print content:4
Thread-6 thread print content:5

Thread-7 thread print content:6
Thread-8 thread print content:7
Thread-9 thread print content:8

Thread-10 thread print content:9

```

从输出中也可以发现，同一时刻至多有$3$个线程拥有打印机的执行权。

---

#### Queue

*queue*模块下提供了常见的线程安全队列，比如先进先出(*FIFO*)、后进先出(*LIFO*)、优先级队列等，不过它们基本使用方式大都类似:

```python
import multiprocessing
import queue
import threading

broker = queue.Queue()

def worker():
    while 1:
        item = broker.get()
        if item is None:
            break
        print("{}".format(item))
        broker.task_done()

threads = []
for i in range(multiprocessing.cpu_count()):
    t = threading.Thread(target=worker)
    threads.append(t)
    t.start()

# simulate producer produce data
for i in range(20):
    broker.put(i)

# block until all task are done
broker.join()

# stop workers
for _ in threads:
    broker.put(None)

for t in threads:
    t.join()
```

---

#### 线程池

线程创建、销毁存在一定的开销，很多时候是通过采用线程池的方式来避免了这部分开销。标准库中同样提供了线程池供开发者使用，其接口与进程池接口一致:

```python
from multiprocessing.pool import ThreadPool

def multiply(a, b):
    return a * b

with ThreadPool(4) as thread_pool:
    tasks = []
    for i in range(1, 10):
        for j in range(10, 20):
            tasks.append((i, j))

    output = thread_pool.starmap_async(multiply, tasks)
    print(output.get())

```

---

### 杂言

*threading*模块中*Lock*、*Event*本文并没有提及，主要原因在于其使用方式过于简单了。



#### 参考资料

[queue](https://docs.python.org/3/library/queue.html)

[threading](https://docs.python.org/3/library/threading.html#module-threading)

[GIL](https://realpython.com/python-gil/)

