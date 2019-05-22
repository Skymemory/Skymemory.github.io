---
layout:     post
title:      "Python之描述符工作机制"
date:       2018-10-27 22:30:00 +0800
author:     "Sky丶Memory"
header-img: "img/2018-10-27-01-bg.jpeg"
tags:
    - Python
---

描述符，算得上Python语言层次上面一个高级的特性，这篇文章会从三个方面介绍描述符特性:定义、触发机制、应用例子。另外，本文中所有代码均在Python 3.4下进行测试

---

#### 定义

Python标准语法规定描述符需要实现特定的描述符协议，其具体的描述符协议对应如下:

```python

object.__get__(self, obj, type=None) --> value

object.__set__(self, obj, value) --> None

object.__delete__(self, obj) --> None

```

也就是说，如果一个对象定义了任意上述3个方法，则该对象被视为描述符对象，Python解释器会在一定条件下自动触发描述符行为。

描述符从类别上分为两类:数据描述符和非数据描述符，数据描述符需要定义__get__和__set__，非数据描述符只需定义__get__，__delete__方法的定义与是否为数据描述符无关

数据描述符与非数据描述符之间的差异主要体现在：当实例属性中存在与描述符同名属性时，如果描述符为数据描述符，则数据描述符优先级更高，否则实例属性优先级更高，一个简单对比两者差异的例子：

```python
class DataDescriptor:
    def __init__(self, x):
        self.x = x

    def __get__(self, instance, owner):
        if instance is None:
            return self
        else:
            return self.x

    def __set__(self, instance, value):
        self.x = value


class NonDataDescriptor:
    def __init__(self, x):
        self.x = x

    def __get__(self, instance, owner):
        if instance is None:
            return self
        else:
            return self.x


class Person:
    data_age = DataDescriptor(25)
    non_data_age = NonDataDescriptor(25)


if __name__ == '__main__':
    p = Person()
    print(p.__dict__)

    # test non-data descriptor
    print("Before test non-data descriptor, age is {}".format(p.non_data_age))
    p.__dict__['non_data_age'] = 90
    print("After test non-data descriptor, age is {}".format(p.non_data_age))

    # test data descriptor
    print("Before test data descriptor, age is {}".format(p.data_age))
    p.__dict__['data_age'] = 233
    print("After test data descriptor, age is {}".format(p.data_age))
    p.data_age = 26
    print("After write attr, instance dict is {}, attr value is {}".format(p.__dict__, p.data_age))


# output
{}
Before test non-data descriptor, age is 25
After test non-data descriptor, age is 90
Before test data descriptor, age is 25
After test data descriptor, age is 25
After write attr, instance dict is {'non_data_age': 90, 'data_age': 233}, attr value is 26
```

---

#### 触发机制

清楚了描述符的定义后，那再来看看描述符协议是怎么触发的。目前在Python中有两种方式触发描述符协议:

- 直接调用对象描述符协议方法，比如描述符对象d，则d.__get__(obj, type)会触发描述符协议
- 当作为类属性时，实例或者类访问该属性，会自动触发描述符协议

第一种使用方法比较简单，这里主要介绍第二种方式的触发机制，从一个简单的例子看起：

```python
class Descriptor:
    def __init__(self, name):
        self.name = name

    def __get__(self, instance, owner):
        if instance is None:
            return self
        else:
            return self.name


class Person:
    name = Descriptor('SkyMemory')
```

上述代码中，Person类中name属性为描述符对象，我们知道类属性是可以通过实例或者类访问的，那这两种不同的访问触发机制有什么不同吗？

针对类属性访问，也就是Person.name形式，触发机制存在于type.__getattribute__()，Person.name会被转换为Person.__dict__['name'].__get__(None, Person)方式调用描述符协议，整个触发过程类似如下代码:

```python
def __getattribute__(self, key):
    "Emulate type_getattro() in Objects/typeobject.c"
    v = object.__getattribute__(self, key)
    if hasattr(v, '__get__'):
       return v.__get__(None, self)
    return v
```

另外，类操作描述符的方式，永远不会触发描述符__set__和__delete__方法

针对实例属性访问，也就是Person().name形式，触发机制存在于object.__getattribute__()，p = Person(), p.name会被转换为type(p).__dict__['name'].__get__(p, type(p))方式，这里面还会涉及到数据描述符和非数据描述符优先级的逻辑。


通过对比类和实例访问时的触发机制，我们可以得到一些关于描述符触发机制的关键点:

- 描述符是通过__getattribute__()方法触发，类和实例访问触发时，对应的__get__参数不同
- 数据描述符优先于实例__dict__属性，非数据描述符可以通过实例__dict__属性覆盖

最后，从本质上来说，描述符有点类似于设计模式中的代理模式，我们可以通过描述符将隐藏属性存放于描述符中，对外暴露描述符即可，再由解释器帮我们完成整个代理协议的转换。

---

#### 应用例子

##### property

property是Python提供的一种快速创建数据描述符的内置函数，也是比较常用的一种创建描述符方式，使用方式有如下两种:

```python
# first
class C:
    def __init__(self):
        self.__x = None

    @property
    def nx(self):
        return self.__x

    @nx.setter
    def nx(self, value):
        self.__x = value

    @nx.deleter
    def nx(self):
        del self.__x


# second

class D:
    def __init__(self):
        self.__x = None

    def getx(self):
        return self.__x

    def setx(self, value):
        self.__x = value

    def delx(self):
        del self.__x

    xx = property(getx, setx, delx, "I'm the 'x' property")

```

property本身也是一个描述符，不过它额外提供了一些公共接口，这里有个property等价实现:

```python
class property(object):
    "Emulate PyProperty_Type() in Objects/descrobject.c"

    def __init__(self, fget=None, fset=None, fdel=None, doc=None):
        self.fget = fget
        self.fset = fset
        self.fdel = fdel
        if doc is None and fget is not None:
            doc = fget.__doc__
        self.__doc__ = doc

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        if self.fget is None:
            raise AttributeError("unreadable attribute")
        return self.fget(obj)

    def __set__(self, obj, value):
        if self.fset is None:
            raise AttributeError("can't set attribute")
        self.fset(obj, value)

    def __delete__(self, obj):
        if self.fdel is None:
            raise AttributeError("can't delete attribute")
        self.fdel(obj)

    def getter(self, fget):
        return type(self)(fget, self.fset, self.fdel, self.__doc__)

    def setter(self, fset):
        return type(self)(self.fget, fset, self.fdel, self.__doc__)

    def deleter(self, fdel):
        return type(self)(self.fget, self.fset, fdel, self.__doc__)

```

##### 函数和方法

先从一个例子看起:

```python
>>> class D(object):
     def f(self, x):
          return x

>>> d = D()
>>> D.__dict__['f'] # Stored internally as a function
<function f at 0x00C45070>
>>> D.f             # Get from a class becomes an unbound method
<unbound method D.f>
>>> d.f             # Get from an instance becomes a bound method
<bound method D.f of <__main__.D object at 0x00B18C90>>
```

上述输出中产生了一个奇怪的问题：D属性__dict__中存放的f类型是函数，而通过类或者实例访问，则变成了方法

f自身定义为在命名空间D下的一个函数，但通过属性或者类的访问方式，f类型发生了变化，结合上面讲述的描述符，容易生成一种猜想，Python的函数对象是不是也是一种描述符？简单的验证下猜想：
```python
def test():
    pass

protocols = ['__set__', '__get__', '__delete__']
print(filter(protocols.__contains__, dir(test)))

# output
['__get__']

```

从上述的输出结果中，我们可以得知，Python的函数对象是非数据描述符，这也就解释了为什么通过类或者实例访问时，f类型会产生变化。

关于函数对象的__get__方法实现，这里有个参考的实现:
```python
import types

class Function(object):
    . . .
    def __get__(self, obj, objtype=None):
        "Simulate func_descr_get() in Objects/funcobject.c"
        return types.MethodType(self, obj, objtype)
```

##### 静态方法和类方法

Python中有两个比较常用的内置函数:classmethod和staticmethod，staticmethod用于缩短代码垂直距离，classmethod用于实现类的多态性，而这两个方法的背后，都用到了描述符。

关于staticmethod的实现，其大概的一种等价实现方式:
```python
class staticmethod:
    def __init__(self, func):
        self.func = func

    def __get__(self, instance, owner):
        return self.func

```
而classmethod的等价实现方式也与此类似:

```python
from functools import partial


class myclassmethod:
    def __init__(self, func):
        self.func = func

    def __get__(self, instance, owner):
        if owner is None:
            owner = type(instance)
        return partial(self.func, owner)

```

另外，关于类方法或静态方法上存在多个装饰器时，@classmethod和@staticmethod一定要放在最上面的原因，这里也给出了对应的答案。

[Descriptor HowTo Guide](https://docs.python.org/3.4/howto/descriptor.html?highlight=descriptor)