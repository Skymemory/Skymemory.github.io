---
layout:     post
title:      "Django之signal"
date:       2019-01-01 19:30:00 +0800
author:     "Sky丶Memory"
header-img: "img/2019-01-01-01-bg.jpeg"
tags:
    - Django
    - signal

---

#### 背景

在实际的开发场景中，常常会遇到这样一类需求：服务A产生了某种类型的事件(比如用户注册、订单状态变更等)，其他依赖服务需要感知到该事件的发生，一般典型的做法就是服务A将对应的事件发布到某个topic下，其他需要关注该事件的服务订阅对应的topic即可，俗称Pub/Sub模式。

构建于Django框架之上的项目，往往都存在多个应用，不同应用之间的事件通知就变得很重要了，Signal则是Django官方提供的一种事件管理器来帮助开发者更好的解藕事件发送方和事件接收方。

#### 初窥

Signal类定义在Django源代码下dispatch/dispatcher.py模块中，对外提供5个方法：

1. Signal.connect(*receiver*, *sender=None*, *weak=True*, *dispatch_uid=None*)

2. Signal.disconnect(*receiver=None*, *sender=None*, *dispatch_uid=None*)

3. Signal.send(*sender*, **\*kwargs*)

4. Signal.send_robust(*sender*, **\*kwargs*)

5. Signal.has_listeners(self, sender=None)

1、2方法可看成是一个对称的接口，主要用于连接、断开接收方；3、4方法用于发送方发送事件通知，两者的区别在于向某个接收方发送事件时，如果接收方处理对应的事件出现未捕获的异常，send方法继续向上抛出，而send_robust捕获异常并继续通知剩余的接收方；5方法见名知意。。。。

了解了基本的接口使用，再来看看Signal是怎么做路由的。

通常情况下，接收方只关注自己关心的事件，这在接收方注册到信号实例上可明确指出，稍微偷窥一下这部分实现的源代码:

```python
def connect(self, receiver, sender=None, weak=True, dispatch_uid=None):
    """
    Connect receiver to sender for signal.

    Arguments:

        receiver
            A function or an instance method which is to receive signals.
            Receivers must be hashable objects.

            If weak is True, then receiver must be weak referenceable.

            Receivers must be able to accept keyword arguments.

            If a receiver is connected with a dispatch_uid argument, it
            will not be added if another receiver was already connected
            with that dispatch_uid.

        sender
            The sender to which the receiver should respond. Must either be
            a Python object, or None to receive events from any sender.

        weak
            Whether to use weak references to the receiver. By default, the
            module will attempt to use weak references to the receiver
            objects. If this parameter is false, then strong references will
            be used.

        dispatch_uid
            An identifier used to uniquely identify a particular instance of
            a receiver. This will usually be a string, though it may be
            anything hashable.
    """
    from django.conf import settings

    # If DEBUG is on, check that we got a good receiver
    if settings.configured and settings.DEBUG:
        assert callable(receiver), "Signal receivers must be callable."

        # Check for **kwargs
        if not func_accepts_kwargs(receiver):
            raise ValueError("Signal receivers must accept keyword arguments (**kwargs).")

    if dispatch_uid:
        lookup_key = (dispatch_uid, _make_id(sender))
    else:
        lookup_key = (_make_id(receiver), _make_id(sender))

    if weak:
        ref = weakref.ref
        receiver_object = receiver
        # Check for bound methods
        if hasattr(receiver, '__self__') and hasattr(receiver, '__func__'):
            ref = WeakMethod
            receiver_object = receiver.__self__
        if six.PY3:
            receiver = ref(receiver)
            weakref.finalize(receiver_object, self._remove_receiver)
        else:
            receiver = ref(receiver, self._remove_receiver)

    with self.lock:
        self._clear_dead_receivers()
        for r_key, _ in self.receivers:
            if r_key == lookup_key:
                break
        else:
            self.receivers.append((lookup_key, receiver))
        self.sender_receivers_cache.clear()

```



从源代码中我们可以看出，Signal会为每一个receiver计算一个lookup_key，通过lookup_key标识对应的receiver，**并且保证接收方唯一** 。

这大概就是接收方注册时，Signal内部做的事情；那再看看发送方发送时，Signal怎么做的路由。

```python
def send(self, sender, **named):
    if not self.receivers or self.sender_receivers_cache.get(sender) is NO_RECEIVERS:
        return []

    return [
        (receiver, receiver(signal=self, sender=sender, **named))
        for receiver in self._live_receivers(sender)
    ]


def _live_receivers(self, sender):
    receivers = None

    if self.use_caching and not self._dead_receivers:
        receivers = self.sender_receivers_cache.get(sender)
        if receivers is NO_RECEIVERS:
            return []
    if receivers is None:
        with self.lock:
            self._clear_dead_receivers()
            senderkey = _make_id(sender)
            receivers = []
            for (receiverkey, r_senderkey), receiver in self.receivers:
                if r_senderkey == NONE_ID or r_senderkey == senderkey:
                    receivers.append(receiver)
            if self.use_caching:
                if not receivers:
                    self.sender_receivers_cache[sender] = NO_RECEIVERS
                else:
                    self.sender_receivers_cache[sender] = receivers
    non_weak_receivers = []
    for receiver in receivers:
        if isinstance(receiver, weakref.ReferenceType):
            receiver = receiver()
            if receiver is not None:
                non_weak_receivers.append(receiver)
        else:
            non_weak_receivers.append(receiver)
    return non_weak_receivers
```

从_live_receiver方法中，很容易看出路由规则：比较接收方与发送方对应的lookup_key是否相同即可(例外情况：接受方注册时可以不明确指定sender，意味着它接受所有的事件通知，也就是上述源码中的**r_senderkey == NONE_ID** )

另外，Django内部提供了一组内建信号，便于业务层接受框架层的事件通知，支持的[内建信号列表](https://docs.djangoproject.com/en/1.11/ref/signals/)



#### 使用

这里我就不举使用例子了，简单说一下Signal的使用实践:

- APP目录下signals.py模块定义信号
- APP目录下的apps.py文件，在AppConfig子类ready方法中注册接受方，不然有可能会收不到对应的事件
- send方法是同步调用

#### 结束语

刚开始看Signal时，理解起来并没有想象中的那么顺，后面理解了才发现原因：之前使用的Pub/Sub模式都是**Pull模型** ,而Signal使用的却是**Push模型**，两者之间的差异在于**Push模型可以做调度** 