---
layout: post-math
title: 密码学中的椭圆曲线分类
date: 2025-03-01 21:00:00 +0800
tags: 读书笔记
render_with_liquid: false
---

椭圆曲线是密码学中非常重要的部分，由于经典的离散对数问题所衍生的算法暴露出了越来越多的缺陷，椭圆曲线对数问题成为了目前公钥加密的最佳选择。

要理解椭圆曲线密码学，首先需要知道椭圆曲线的分类，在此简单总结。其他人的总结可以参考：

- [一场椭圆曲线的寻根问祖之旅](https://www.infoq.cn/article/LbO7iMFxAWd6uM6A6QY8)
- [ed25519加密签名算法及应用](https://zhuanlan.zhihu.com/p/579276183)
- [椭圆曲线数据库](https://neuromancer.sk/std/)

# 有限域

密码学中一般研究有限域，实际上就是研究两种有限域

- 素数域$$\mathbb F_p$$，和整数的区别就是加减乘除需要模一个素数。
- 二进制拓展域$$\mathbb F_{2^m}$$，实际上对应了有限次数的二进制多项式，加减法等同于异或，而乘法需要使用[无进位乘法](https://en.wikipedia.org/wiki/Carry-less_product)。

具体使用时，二进制拓展域一般还需要再指定一个不可约多项式，每次运算都要模这个多项式。由于二进制拓展域所用的无进位乘法比较特殊，所以用得比较少。

# Weierstrass 曲线

这是最通用的椭圆曲线形式，形为

$$
y^2=x^3+ax+b
$$

另外，椭圆曲线还需要定义对应的有限域 $$F$$ 以及一个合适的基点 $$G$$。随意选择的椭圆曲线的参数很可能根本不安全，因此一般都会使用公认的曲线。

椭圆曲线的加法运算都有一个统一的定义，即A+B定义为过A和B的割线和曲线相交的点。在使用 Weierstrass 形式时，需要区分$$A=B$$（切线）、$$A\neq B$$（割线）、$$A=-B$$（不相交）三种情况，比较繁琐，因此新的方法一般不用此形式。

## 命名曲线

SECG 命名了大部分常见的 Weierstrass 形式的曲线，命名为[`sec[p/t][xxx][r/k]N`](https://www.secg.org/SEC2-Ver-1.0.pdf)，其中 sec 是 Standards for Efficient Cryptology 的缩写，p/t 表示质数域/二进制拓展域，xxx 是曲线的位数，r/k表示随机参数/**Koblitz 曲线**。Koblitz 曲线参数是比较简洁的小整数，并且具有利于计算的特性。

NIST 同样命名了大量的曲线，尤其是 P-256（secp256r1）是最常用的椭圆曲线，但因为随机数选取不透明而且可能有后门（[Dual_EC_DRBG](https://en.wikipedia.org/wiki/Dual_EC_DRBG)），现在更推荐用其他的曲线。

# Montgomery 曲线和扭曲 Edwards 曲线

一些特殊的曲线不但可以使用 Weierstrass 形式表示，还能使用 Montgomery 或者是扭曲 Edwards 形式表示，这两种形式是等价的。

Montgomery 形式为

$$
By^2=x^3+Ax^2+x
$$

而扭曲 Edwards 形式为

$$
ax^2+y^2=1+dx^2y^2
$$

“扭曲”指的是 $$a\neq 1$$ 的情况，因为 Edwards 形式要求 $$a=1$$。

常用的 Montgomery 曲线是 [Curve25519](https://neuromancer.sk/std/other/Curve25519)，所在的素数域中 $$p=2^{255}-19$$，对应的扭曲 Edwards 曲线是 [Ed25519](https://neuromancer.sk/std/other/Ed25519)，对应的密钥交换函数则称为X25519。如果255位不能满足需求，Curve448是对应448位的版本。

## 区别

这两种形式的曲线的区别是（[Why Curve25519 for encryption but Ed25519 for signatures?](https://crypto.stackexchange.com/questions/27866/why-curve25519-for-encryption-but-ed25519-for-signatures)）：
- Montgomery 曲线的**变基**标量乘法（$$a*P$$）很高效，因此适合于密钥交换（ECDH）。
- 扭曲 Edwards 曲线的**定基**（$$a*G$$）或者是**双基**（$$a*P+b*Q$$）标量乘法很高效，因此适合于数字签名（EdDSA）。

# 计算

虽然椭圆曲线的计算公式对于有理数和有限域的封闭的，但是运算过程中需要计算除法，也就是求一个数的逆元。而在质数域中，求逆基本只能通过 $$p-2$$ 次幂计算，需要进行上百次乘法，非常复杂。因此，椭圆曲线相关运算都需要特殊的公式用于加速，这在 Bernstein 的 [Explicit-Formulas Database](https://hyperelliptic.org/EFD/index.html) 中可以查到。
