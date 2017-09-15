---
layout: post
date:   2017-09-09 22:20:38
categories: 学习笔记
tags:   crytpo ctf rsa
---

## 壹、中国剩余定理与RSA

### 一、首先简述中国剩余定理（chinese remainder theorem，CRT）

该定理非常方便地用于求解同余方程组，还是喜欢古书中的题目：

```
有物不知其数，三三数之剩二，五五数之剩三，七七数之剩二。问物几何？
抽象成数学符号就是：
x mod 3 = 2  |  x mod m1 = a1
x mod 5 = 3  |  x mod m2 = a2
x mod 7 = 2  |  x mod m3 = a3
```

小学时候参加数学竞赛做过，但是一直不知道是怎么回事，该怎么求解。中国剩余定理解法有二，一是单基数转换法(Single-Radex Conversion, SRC)，另一个是混合基数转换法(Mixed-Radex Conversion, MRC)

#### 1、SRC法
这个方法的证明还比较简单，这里暂不给出，这里给出公式（SRC）：（m之间默认应该是互质的）

```
M = m1 * m2 * m3 * ... * mn
Mi = M / mi
inv_Mi 是Mi在模mi下的逆元

所以：
x = a1*M1*inv_M1 + a2*M2*inv_M2 + ... + an*Mn*inv_Mn mod M
```

概括成一句诗就是“三人同行七十稀，五树梅花廿一支，七子团圆正半月，除百零五使得知”

意思是：将除以3得到的余数乘以70，将除以5得到的余数乘以21，将除以7得到的余数乘以15，全部加起来后减去105（或者105的倍数），得到的余数就是答案。

其中70是因为 35 mod 3 = 2 (21 mod 5 = 1; 15 mod 7 = 1)

#### 2、MRC法和MMRC法（应用于RSA中）

这种方法在看起来比较复杂了一点，但是由于其递归性质，对计算机编程以及提高计算速度很友好。这里假设mi都是素数，所以mi用pi表示。

```
Bji = inv(pj) mod pi (pj在pi中的逆元)
MMRC计算如下三角式：
a11
a21 a22
a31 a32 a33
……
ai1 = ai (mod pi) (各个同余式的余数)
ai(j+1) = (aij - ajj)* Bji (mod pi), 1<=j < i

唯一解为：
x = a11 + a22 * p1 + a33 * p1*p2 + ... + aii * p1*p2*..*p(i-1)
```
同样，以上述题目为例，计算：
```
B j| 1 | 2 | 3 |
i 1| \   #   #  
  2| 2   \   # 
  3| 5   3   \

aii系列矩阵：
a11 = a1 = 2 (mod 3)
a21 = a2 = 3 (mod 5)
a22 = (a21-a11)*B12 = (3-2)*2 = 2 (mod 5)
a31 = a3 = 2 (mod 7)
a32 = (a31-a11)*B13 = (2-2)*5 = 0 (mod 7)
a33 = (a32-a22)*B23 = (0-2)*3 = 1 (mod 7)

2
3 2
2 0 1

x = a11 + a22 * p1 + a33 * p1*p2
  =  2  +  2  * 3  +  1  * 3 * 5
  =  2 + 6 + 15
  = 23 (mod 105)  
```

### 二、中国剩余定理在RSA中的使用

该定理已经确定应用于openssl中，[在privkey中已经使用相关的参数](https://cryptography.io/en/latest/hazmat/primitives/asymmetric/rsa/#cryptography.hazmat.primitives.asymmetric.rsa.RSAPrivateNumbers.dmp1)。
是使用的MMRC的方式。求解`m = c ** d (mod p*q)`等价于求方程组`m1 = c ** d (mod p); m2 = c ** d (mod q);`，同理，d在p，q中的映射为`d1 = d mod Phi(p) = d mod (p-1); d2 = d mod (q-1);`

所以在privkey中实际保存的数据有：
```
...
dmp1 = d mod p-1
dmq1 = d mod q-1
iqmp = inv(q) mod p
...
```
计算时，就是这样（由于是iqmp，所以mod q作为①式，注意后面pq的位置）
```
c ** dmq1 = m2 (mod q)
c ** dmp1 = m1 (mod p)

B j| 1 | 2 |
i 1| \   #
  2|iqmp \ 

a11 = m2 (mod q)
a21 = m1 (mod p)
a22 = (m1-m2)*iqmp (mod p)

m = a11 + a22 * p1
  = m2 + (m1 - m2) * iqmp * q
```
由于较低的指数和模数，所以能提高不少计算速度。

### 三、针对中国剩余定理的出错攻击
知道了原理，那么攻击解释起来就非常简单了。

注意上一节中的m1，如果在计算m1的时候出错，我们得到一个错误的消息m'，那么（m-m’）是q的倍数，辗转相除法计算他和N的最大公约数就能够得到q，从而将N分解。

### 四、一些解题技巧
#### 1、低底数获取N
[ESPer](https://github.com/TokyoWesterns/twctf-2016-problems/tree/master/ESPer)这题中并没有直接给出N，但是允许加密任意数据。那么就分别加密2和3，分别得到c2,c3，那么计算`gcd(2 ** e - c2, 3 ** e - c3)`就能够得到N，呃通常为0x10001。

#### 2、N和Phi(N)
N和Phi(N)是比较接近的，已知一个可以通过小范围内爆破得到另一个。


