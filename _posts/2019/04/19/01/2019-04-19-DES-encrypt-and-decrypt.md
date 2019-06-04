---
layout:     post
title:      "DES加密算法原理及其实现"
date:       2019-04-19 15:00:00 +0800
author:     "Sky丶Memory"
header-img: "img/2019/04/19/01/bg.jpg"
tags: Cryptology
usemathjax: true

---

*DES*算法全称*Data Encryption Standard*，即数据加密标准，是一种对称加密算法。

#### DES加密原理

*DES*算法通过将明文分成大小为*8*字节的多个明文块（不足*8*字节通过填充方式补齐），然后对每个明文块应用密钥加密得到对应的密文块，将多个密文块组合起来就得到了最终的密文，其过程如下图所示：

![](/img/2019/04/19/01/01.png)

明文块转换为密文块流程：

![](/img/2019/04/19/01/02.gif)

明文块如何转换为密文块是整个*DES*算法的核心，其步骤可以分为两步：

- 根据密钥计算出子密钥
- 子密钥与明文通过一系列的迭代操作，变成密文的加密过程

在开始后续了解之前，先编写下面代码，后面会在*DES*类中添加方法。

```python
def nth_bit_of_byte(v, n):
    return v >> n & 1
    
class DES:
    def __init__(self, secret_key):
        assert isinstance(secret_key, bytes) and len(secret_key) == 8
        self.secret_key = secret_key
        # 计算子密钥
        self.keys = list(self._calc_subkeys())
```



下面，我们分别来了解下子密钥和加密过程。

---

#### 子密钥

*DES*的初始密钥***K***为*64*位，按*8*行*8*列从左到右、从上到下排列，采用字节内高位低编号、低址优先的编号方式，以密钥*abcdefgh*为例，其对应的二进制矩阵为:

![](/img/2019/04/19/01/03.jpg)

*bit*编号方式为以*1*开始，从左到右、从上到下递增，这在密钥置换中会使用到。

首先，将初始密钥***K***经过***PC1***置换，生成*56*位的串。

![img](/img/2019/04/19/01/04.png)

从上表中可知，初始密钥***K***第*8*、*16*、*24*、*32*、*40*、*48*、*56*、*64*共*8*个位被去掉，被去掉的*8*个位一般用来做奇偶性校验，以防不同***K***经过***PC1***变换后得到的串相同，不过实际中很少校验。

经过***PC1***置换后，将输出分为前*28*位 *$$C_{0}$$*、后*28*位*$$D_{0}$$*，分别循环左移*1*位得到*$$C_{1}$$*、*$$D_{1}$$*，将*$$C_{1}$$*、*$$D_{1}$$*合并为*56*位串，经***PC2***变换得到子密钥***$$K_{1}$$***。*$$C_{1}$$*、*$$D_{1}$$*分别循环1位得到*$$C_{2}$$*、*$$D_{2}$$*，合并*$$C_{2}$$*、*$$D_{2}$$*，经***PC2***变换得到子密钥*$$K_{2}$$*。重复此过程，直到得到子密钥*$$K_{16}$$*。

![](/img/2019/04/19/01/05.png)

循环左移位数与迭代轮数之间的对应关系：

![](/img/2019/04/19/01/06.png)

计算子密钥的代码：

```python
    def _calc_subkeys(self):
        k = bytearray(7 * 8)
        # 密钥置换PC1
        for i, v in enumerate(PC1):
            q, r = (v - 1) >> 3, (v - 1) & 0x07
            k[i] = nth_bit_of_byte(self.secret_key[q], 7 - r)
        # 循环左移、PC2变换
        for i in ROTATE_STEPS:
            k[:28] = k[i:28] + k[:i]
            k[28:] = k[28 + i:] + k[28: 28 + i]
            yield bytearray(map(lambda x: k[x - 1], PC2))
```

其中变量*PC1*、*PC2*、*ROTATE_STEPS*分别对应上述中的***PC1***、***PC2***及循环位移序列。

---

#### 加密过程

*64*位明文串经过初始置换***IP***，产生了新*64*位输出，将其分为左*32*位*$$L_{0}$$*、右*32*位*$$R_{0}$$*。

![](/img/2019/04/19/01/07.png)

将*$$R_{0}$$*与子密钥*$$K_{1}$$*经过函数*F（$$R_{0}$$，$$K_{1}$$）*得到*32*位输出*$$f_{1}$$*，*$$f_{1}$$*与*$$L_{0}$$*做二进制异或运算结果赋值给*$$R_{1}$$*，*$$R_{0}$$*则赋值给*$$L_{1}$$*。然后*$$R_{1}$$*与子密钥*$$K_{2}$$*经过函数*F（$$R_{1}$$，$$K_{2}$$）*得到*32*位输出*$$f_{1}$$*，*$$f_{1}$$*与*$$L_{1}$$*做二进制异或运算结果赋值给*$$R_{2}$$*，*$$R_{1}$$*则赋值给*$$L_{2}$$*，重复此过程*16*轮，得到*$$L_{16}$$*、*$$R_{16}$$*。

*$$L_{16}$$*、*$$R_{16}$$*合并后的结果经过置换***$$IP^{-1}$$***，得到最终密文。

![](/img/2019/04/19/01/08.png)

---

#### 函数F

***F***的作用在于将*32*位数据*d*和*48*子密钥*k*进行一系列计算，产生*32*位输出数据。

首先，*d*通过扩展置换*E*从*32*变为*48*位，将扩展置换后的结果与子密钥*k*进行二进制异或运算得到*t*。

![](/img/2019/04/19/01/09.png)

将*t*分成*8*个*6*位的块，每块通过对应的一个***S***盒产生一个*4*位输出，***S***盒具体处理逻辑为：取第一位和第六位得到二进制数*r*，第二位到第五位构成另外一个二进制数*c*，查询***S***盒*r*行*c*列得到对应的十进制数，将该十进制数转换为*4*位二进制数输出。

![](/img/2019/04/19/10.jpg)

*8*个块对应的***S***盒输出拼接后，通过***P***盒置换，产生函数***F***输出。

![](/img/2019/04/19/04/11.png)



整个加密过程涉及代码：

```python
    def _f(self, d, k):
        # 扩展置换
        s1 = map(lambda x: d[x - 1], E)
        # 异或运算
        s = list(map(operator.xor, k, s1))

        t = bytearray(32)
        # S盒置换
        for i in range(0, 48, 6):
            row = s[i] << 1 | s[i + 5]
            col = s[i + 1] << 3 | s[i + 2] << 2 | s[i + 3] << 1 | s[i + 4]
            v = S_BOX[i // 6 * 64 + row * 16 + col]
            offset = i // 6 * 4
            for j in range(4):
                t[offset + j] = nth_bit_of_byte(v, 3 - j)

        # P盒置换
        tt = bytearray(map(lambda x: t[x - 1], P_BOX))
        return tt

    def _bits_to_bytes(self, bits, rs, n):
        for i in range(n):
            q, r = i >> 3, i & 0x07
            rs[q] |= bits[i] << (7 - r)
        return rs

    def encrypt(self, clear_text):
        s = bytearray(64)
        # 初始置换
        for i, v in enumerate(IP):
            q, r = (v - 1) >> 3, (v - 1) & 0x07
            s[i] = nth_bit_of_byte(clear_text[q], 7 - r)

        # 迭代运算
        s1, s2 = s[:32], s[32:]
        for k in self.keys:
            f1 = self._f(s2, k)
            t = bytearray(map(operator.xor, s1, f1))
            s1, s2 = s2, t

        # 末置换
        r = s2 + s1
        rr = bytearray(map(lambda x: r[x - 1], INVERSE_IP))

        return self._bits_to_bytes(rr, bytearray(8), 64)
```

---

#### 其他

解密的过程亦为加密的逆运算，不做过多解释。

```python
    def decrypt(self, secret_text):
        s = bytearray(64)
        for i, v in enumerate(INVERSE_IP):
            q, r = i >> 3, i & 0x07
            s[v - 1] = nth_bit_of_byte(secret_text[q], 7 - r)

        s1, s2 = s[32:], s[:32]
        for k in reversed(self.keys):
            f1 = self._f(s1, k)
            t = bytearray(map(operator.xor, s2, f1))
            s1, s2 = t, s1

        s = s1 + s2
        r = bytearray(64)
        for i, v in enumerate(IP):
            r[v - 1] = s[i]

        return self._bits_to_bytes(r, bytearray(8), 64)
```

另外，本文有个地方没有涉及到的是填充这部分，后续有时间在补上。



#### 参考文献

[DES加密算法原理解析](<https://marshal-r.iteye.com/blog/2089963>)