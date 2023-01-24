---
layout: post-math
title: Hu矩观后感
date: 2020-12-03 21:46:36 +0800
tags: 读书笔记
---

参考资料：

- M. K. Hu, "Visual Pattern Recognition by Moment Invariants", IRE Trans. Info. Theory, vol. IT-8, pp.179–187, 1962
- J. Flusser: "[On the Independence of Rotation Moment Invariants](http://library.utia.cas.cz/prace/20000033.pdf)", Pattern Recognition, vol. 33, pp. 1405–1410, 2000.
- J. Flusser and T. Suk, "[Rotation Moment Invariants for Recognition of Symmetric Objects](http://library.utia.cas.cz/separaty/historie/flusser-rotation%20moment%20invariants%20for%20recognition%20of%20symmetric%20objects.pdf)", IEEE Trans. Image Proc., vol. 15, pp. 3784–3790, 2006.

# 导论

昨晚概率论刚刚上到矩（moment），正好之前opencv的时候概念又看到过非常神秘的Hu矩（Hu moments）即8个由3阶以内中心矩构成的（平移、缩放和）旋转不变量。今天无聊时维基搜了一波，发现出乎意料地有趣。

Flusser的论文中，利用了复数，很轻松地便证明了矩的旋转不变性（复数nb！），并指出了Hu矩中第三组矩是冗余的，且在三阶以内有8个旋转不变量，Hu矩去掉冗余的一个后还少了一个。

# 矩

概率论中也有矩，设随机变量$$X$$的概率密度函数为$$f(x)$$，则n阶原点矩为

$$
\mu_n = \int_{-\infty}^\infty x^n f(x)\mathrm d x
$$

在图像处理中，矩的定义也是类似的，设图像为$$f(x,y)$$

$$
M_{pq}=\int_{-\infty}^\infty\int_{-\infty}^\infty (x-\bar x)^p(y-\bar y)^q f(x,y) \mathrm d x \mathrm d y
$$

并且一般把$$p+q$$叫做矩的阶数。只不过通常用到的是中心矩，这样得到的矩和图像的位置无关

$$
\mu_{pq}=\int_{-\infty}^\infty\int_{-\infty}^\infty (x-\bar x)^p(y-\bar y)^q f(x,y)\mathrm d x \mathrm d y
$$

而且还是归一化的中心矩，这样使得矩和图像的大小也无关

$$
\eta_{pq}=\frac {\mu_{pq}} {\mu_{00}^{(1+\frac {i+j} 2)}}
$$

接下来的问题是，能不能找到什么方法，使得矩和旋转也无关呢？

# 复矩

Flusser的方法非常地有趣，他先是定义了复矩（complex moment）

$$
c_{pq}=\int_{-\infty}^\infty\int_{-\infty}^\infty (x+\mathrm iy)^p(x-\mathrm iy)^qf(x,y)\mathrm dx \mathrm dy
$$

其中$$\mathrm i$$为虚数单位。复矩显然是普通的矩的多项式，利用二项式定理可以计算，不多解释。

如果令$$z=x+\mathrm iy$$，且$$g(z)=f(x,y)$$，则可以改写为

$$
c_{pq}=\int_{-\infty}^\infty\int_{-\infty}^\infty z^p\bar z^q g(z)\mathrm dS
$$

显然可以看出，p和q交换后结果为原来的共轭，即$$c_{pq}=\overline {c_{qp}}$$。如果$$p=q$$，则结果为实数，否则结果通常应该是复数，为了获得实数的结果，可以单独取实部和虚部。

## 旋转

在复数的情况下，旋转的定义非常简单，$$z$$绕原点逆时针旋转$$\theta$$角的结果是

$$
z^\prime = \mathrm e^{\mathrm i\theta} z
$$

把$$g(z)$$旋转$$\theta$$角，则新的复矩

$$
\begin{align*}
c^\prime_{pq}
&=\int_{-\infty}^\infty\int_{-\infty}^\infty z^p \bar z^q g(\mathrm e^{\mathrm i\theta}z) \mathrm dS \\
&=\mathrm e^{\mathrm i(q-p)\theta}c_{pq}
\end{align*}
$$

显然，对于若干个复矩的乘积，只要$$p-q$$一项加起来是0，则得到的乘积就是旋转不变量。

三阶及以下的旋转不变量的一组基（以乘法可以组成别的不变量）包括：

$$
\begin{align*}
\psi_1&=c_{11}\\
\psi_2&=c_{21}c_{12}\\
\psi_3+\mathrm i\psi_4&=c_{20}c_{12}^2\\
\psi_5+\mathrm i\psi_6&=c_{30}c_{12}^3
\end{align*}
$$

其中后两行的结果通常为复数，前两行的结果保证为实数。$$c_{12}$$也可以是$$c_{01}$$，只要$$q-p=1$$即可。

一阶时$$c_{01}=0$$，没有意义。

## 完备性

当$$c_{12}c_{21}\neq0$$时，可以证明别的不变量可以由他们组成，如

$$
\begin{align*}
c_{20}c_{02}
&=\frac{c_{12}^2c_{21}^2c_{20}c_{02}}{c_{21}^2c_{12}^2}\\
&=\frac{|c_{20}c_{12}|^2}{(c_{21}c_{12})^2}\\
&=\frac{\psi_3^2+\psi_4^2}{\psi_2^2}
\end{align*}
$$

同样的，

$$
c_{30}c_{03}=
\frac{c_{12}^3c_{21}^3c_{03}c_{30}}{c_{12}^3c_{21}^3}=
\frac{\psi_5^2+\psi_6^2}{\psi_2^3}
$$

## Hu矩

至此，可以发现，Hu的那组不变量其实并不是互相独立的。

$$
\begin{align*}
I_1&=\psi_1\\
I_2&=\frac{\psi_3^2+\psi_4^2}{\psi_2^2}\\
I_3&=\frac{\psi_5^2+\psi_6^2}{\psi_2^3}\\
I_4&=\psi_2\\
I_5&=\psi_5\\
I_6&=\psi_3\\
I_7&=\psi_6
\end{align*}
$$

但是根据下面关于旋转对称性的讨论，$$I_2$$和$$I_3$$在图像具有2重或3重旋转对称性时仍为非0值，并不是完全地没有意义。

Flusser的论文中说$$c_{20}c_{12}^2$$和$$c_{30}^2c_{02}^3$$是独立于$$\{I_k\mid k=1\dots7\}$$ 的，但是由上可以看出

$$
\begin{align*}
\mathrm{Re}(c_{20}c_{12}^2)&=\psi_3\\
\mathrm{Im}(c_{20}c_{12}^2)&=\psi_4=\sqrt{I_2I_4^2-I_6^2}\\
c_{30}^2c_{02}^3&=\frac{(c_{30}c_{12}^3)^2\overline{(c_{20}c_{12}^2)}^3}{(c_{21}c_{12})^6}\\
\end{align*}
$$

只不过涉及开方运算，比较复杂罢了。

## 镜像

那么将图像镜像后，获得的这些旋转不变量是否受影响呢？在复数中讨论这个问题也惊人地简单。

$$
\begin{align*}
c^*_{pq}
&=\int_{-\infty}^\infty\int_{-\infty}^\infty z^p \bar z^q \bar g(z) \mathrm dS \\
&=\overline{\int_{-\infty}^\infty\int_{-\infty}^\infty \bar z^p z^q g(z) \mathrm dS} \\
&=\overline {c_{pq}}\\
&=c_{qp}
\end{align*}
$$

也就是说，这些不变量中，实部在镜像变换中不受影响，而虚部在镜像变换中变为相反数。



## 旋转对称性

Flusser的另一篇论文里考虑了旋转对称性，这里复数的好处更是凸显。若$$f(x,y)$$具有N次旋转对称性，即旋转$$2\pi/N$$角度后与原图形重合，则此时非常多的旋转不变量都是0。

即若

$$
g(z)=\mathrm e^{2\pi\mathrm i/N}g(z)
$$

则

$$
c_{pq}=\mathrm \exp(\frac{2\pi\mathrm i}{N}(p-q))\cdot c_{pq}
$$

除非$$p-q$$不是$$N$$的倍数，$$c_{pq}$$总是为0。当要处理的图像具有旋转对称性时，由于之前选取基的时候总是有一项是$$p-q=1$$（这是作者构造这组基的技巧所在），则这组基中大部分的不变量在没有误差的情况下都应该是0！

这说明，如果想要利用矩的不变量来进行分析，一定要注意所选取图像的对称性，若选取的基和Flusser前面选的$$\{\psi_k\mid k=1\dots6)\}$$，那么对于任何具有旋转对称性的图像，所得到的结果除了$$\psi_1$$全是0，实在是非常辣鸡。相比之下，Hu的那组还有很多量非0，仍有判断价值。

