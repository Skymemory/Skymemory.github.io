---
layout:     post
title:      "Python Enum使用"
date:       2018-09-22 12:30:00 +0800
author:     "Sky丶Memory"
header-img: "img/2018-09-22-02-bg.jpg"
tags:
    - Python
    - Enum
---

## 序

在Python 3.4出现之前，语言本身并没有提供关于枚举(Enum)类型，但在实际的工程开发中，大多都会定义一些枚举常量来减少"魔数"的产生，所以不少开发者要求将枚举类型加入Python标准库。

Python官方在[PEP 435](https://www.python.org/dev/peps/pep-0435/)中通过了将枚举类型加入标准库的提案，并从3.4版本起提供enum枚举模块供开发者使用，在3.4之前的版本，可通过第三方库[enum34](https://pypi.org/project/enum34/)使用。

## 定义

首先，简单直观感受下使用方式：
```
import enum


class Animal(enum.Enum):
    dog = 1
    cat = 2
    lion = 3


print("Member name : {}".format(Animal.dog.name))
print("Member value : {}".format(Animal.dog.value))
print("Member type: {}".format(type(Animal.dog)))

print("")

for animal in Animal:
    print("Member name : {}, value : {}".format(animal.name, animal.value))

```
输出：
```
Member name : dog
Member value : 1
Member type: <enum 'Animal'>

Member name : dog, value : 1
Member name : cat, value : 2
Member name : lion, value : 3
```
每个成员都是Animal类型实例，对应有name和value属性，value值允许重复但name不能，Animal类可迭代

---

## 比较

支持标识(identity)和值(value)比较：
```
import enum


class Animal(enum.Enum):
    dog = 1
    cat = 2
    lion = 3


# Comparison using "is"
if Animal.dog is Animal.cat:
    print("Dog and cat are same animals")
else:
    print("Dog and cat are different animals")

# Comparison using "!="
if Animal.lion != Animal.cat:
    print("Lions and cat are different")
else:
    print("Lions and cat are same")

if Animal.dog is Animal(1):
    print("Dog and cat are same animals")
else:
    print("Dog and cat are different animals")

```
输出：
```
Dog and cat are different animals
Lions and cat are different
Dog and cat are same animals
```

一般情况下，更倾向于使用成员的value值做比较，Enum自身支持的比较操作与其内部具体实现(通过元类避免创建同值枚举常量)有关，如果不是太熟悉，建议最好还是将Enum类当作一个"壳"，具体的操作使用方自己决定。

---

## 唯一性

前面提到过，默认情况下Enum类成员value值允许重复，可通过enum模块提供的unique类装饰器约束:
```
import enum


@enum.unique
class Animal(enum.Enum):
    dog = 1
    cat = 2
    lion = 3

    bird = 3

```

输出：
```
Traceback (most recent call last):
  File "demo.py", line 5, in <module>
    class Animal(enum.Enum):
  File ".../.virtualenvs/ss/lib/python3.6/enum.py", line 834, in unique
    (enumeration, alias_details))
ValueError: duplicate values found in <enum 'Animal'>: bird -> lion

```

---

关于标准库中的enum使用，还有其他的一些性质，这里我只罗列了最基本的使用，更具体的使用可参见[PEP 435](https://www.python.org/dev/peps/pep-0435/)