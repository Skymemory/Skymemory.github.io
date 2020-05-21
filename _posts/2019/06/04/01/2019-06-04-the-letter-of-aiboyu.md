---
layout:     post
title:      "艾波寧捎信"
date:       2019-06-04 15:00:00 +0800
author:     "Sky丶Memory"
header-img: "img/2019/06/04/01/bg.jpg"
tags: Math Probability
usemathjax: true


---

这是在*Coursera*学习概率论时的一个作业题目，其中有种模拟解法思路很新颖，这里简单做下记录。

---

#### 问题

巷子呈直線，長L0  = 400 m，艾波寧以v0  = 4 m/s初速等速穿越。士兵時時刻刻瞄準她；第t 秒時是否擊中她，是隨時間t 的均勻的泊松事件(Poisson process)，且與距離無關。其中，平均每μ 秒能擊中一次，μ  = 100 / ln( 50 )  約為25.5622。士兵無法擊中巷子以外的區域；另外，只要她處於巷中，μ 就是常數。 當她每被擊中一槍，速度就會減半；直到她恰中4 槍時，會當場死亡。亦即，中n槍時速度依序為4, 2, 1, 0.5 m/s ，其中n 依序為  0, 1, 2, 3。 

請問艾波寧成功捎信的機率為何?  亦即，在她處於巷子之中時，被射中低於四槍的機率為何?(用小數即可，誤差合理給對) 

---

#### 解法一

根据题意可以获知每秒被击中的概率为$ p=\frac {\ln 50} {100}$，处于不同秒是否被击中是个独立事件， 艾波寧能成功捎信的条件是中枪数小于4，所以我们可以枚举可能的中枪数，然后累加即可。

这里以中1枪为例，假设艾波寧在第*t*秒中枪，已走的距离为$t * 4$，剩下的距离为$400 - t * 4$，艾波寧走出巷子需要的秒数为$seconds = t + \frac {(400 - 4 * t)} 2$，那么在第*t*秒中一枪成功捎信的概率为${(1-p)^{seconds-1}*p}$，通过枚举中一枪可能存在秒数就能计算出艾波寧中一枪成功捎信的概率。

其他中枪数—0、2、3，分析过程与此类似，这里面涉及到的计算量可以通过程序解决，这里提供一种实现方式:

```python
import math


def poisson(k, p):
    return pow(math.e, -p) * pow(p, k) / math.factorial(k)


def main():
    cum, possibility = 0, math.log(50) / 100

    distance = 400
    speeds = (4, 2, 1, 0.5)

    # unhit
    cum = pow(1 - possibility, distance // speeds[0])

    # hit one
    for i in range(1, distance // speeds[0] + 1):
        total_seconds = i + int(
            (distance - i * speeds[0]) / speeds[1]
        )
        cum += pow(1 - possibility, total_seconds - 1) * possibility

    # hit two
    for first_hit_second in range(1, int(distance / speeds[0])):
        left_seconds = (distance - first_hit_second * speeds[0]) // speeds[1]
        for two_hit_second in range(first_hit_second + 1, first_hit_second + left_seconds + 1):
            total_seconds = two_hit_second + int(
                (distance - first_hit_second * speeds[0] - (two_hit_second - first_hit_second) * speeds[1]) / speeds[2]
            )
            cum += pow(1 - possibility, total_seconds - 2) * pow(possibility, 2)

    # hit three
    for first_hit_second in range(1, distance // speeds[0] - 1):
        hit_one_left_seconds = (distance - first_hit_second * speeds[0]) // speeds[1]
        for two_hit_second in range(first_hit_second + 1, first_hit_second + hit_one_left_seconds):
            hit_two_left_second = (distance - first_hit_second * speeds[0] - (two_hit_second - first_hit_second) *
                                   speeds[1]) // speeds[2]
            for three_hit_second in range(two_hit_second + 1, two_hit_second + hit_two_left_second + 1):
                seconds = int(
                    (distance - first_hit_second * speeds[0] - (two_hit_second - first_hit_second) * speeds[1] - (
                            three_hit_second - two_hit_second) * speeds[2]) / speeds[3]) + three_hit_second

                cum += pow(1 - possibility, seconds - 3) * pow(possibility, 3)
    print(round(cum, 2))


if __name__ == '__main__':
    main()
```

---

#### 解法二

第二种解法是根据大数定理得出的，通过计算机模拟多次过程，统计成功捎信所占的比例得出其概率。

```python
import math
import random

def emulate():
    p = math.log(50) / 100
    distance = 400
    speed = 4

    while distance > 0:
        distance -= speed

        # 模拟是否中枪
        if random.random() < p:
            if speed == 0.5:
                return False
            else:
                speed /= 2
    return True

def main():
    times = 100000
    success = 0

    for _ in range(times):
        if emulate():
            success += 1

    print(round(success / times, 2))

if __name__ == '__main__':
    main()
```

最终计算出来结果为0.06