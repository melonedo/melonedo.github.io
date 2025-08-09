---
layout: post-math
title: Swizzle 布局的直观解释和推导
date: 2025-08-04 20:36:00 +0800
tags:
- 编程
- CUDA
render_with_liquid: false
---


上次讲的 swizzle 相关的内容，没来得及展开，今天简单讲讲 swizzle ，顺便学习一下 CUTLASS。

我们知道，英伟达的 GPU 共享内存分为 32 个 bank，每次请求时可以分别从 32 个 bank 中取一个 32 位数据。但是 CUDA 程序每次一个 warp 中 32 个线程可以发出至多 32 个内存请求（去掉重复地址）。如果这些请求都在不同的 bank，则读取一次共享内存即可。如果这些对应的 bank 有重复，则需要访问多次共享内存。如果重复的次数为 N，则称为 N-way bank conflict，需要访问 N 次共享内存。

首先，避免图太大，假设共享内存分 8 个 bank，那么一个 8x8 的矩阵如图，这样的布局在访问连续维度（按行访问）时非常地理想，每次都访问了不同的维度。而访问非连续维度，例如一次访问一行时，所有的访问都发生在同一个 bank，产生 8 路 bank 冲突，即每次访问一列的内容时需要向共享内存请求 8 次，访问效率极其低下。

![swizzle-none](/assets/imgs/swizzle-none.svg)

8x8 矩阵，连续布局。共分 8 个 bank，不同颜色代表不同的 bank。按行访问非常理想，按列访问非常低效。


但是按照不同的维度访问矩阵是非常根本的需求，因此有多种方法解决。一种简单的方法是加一列 padding，把矩阵变成 8x9。


![swizzle-padding](/assets/imgs/swizzle-padding.svg)

8x8 矩阵，加一列 padding 变成 8x9。按行和按列访问都很高效，但需要占用额外空间。


但是这样会占用额外的空间，非常丑陋，怎么办呢？注意到实际上每行实际上还是只有 8 个有效的元素，我们可以把列坐标对 8 取模，保证一行的内容还是放在这一行里，避免溢出到下一行的空间里，还是放到 8x8 的空间里。

也就是说，计算 i 行 j 列地址的公式是`i * 8 + (i + j) % 8`，在前面加 padding 的时候的公式`i * 9 + j = i * 8 + (i + j)`的基础上， 限制列坐标在 8 以内。

![swizzle-circular](/assets/imgs/swizzle-circular.svg)

8x8 矩阵，此处无 padding 胜似有 padding，注意到颜色的分布和上一图完全一致，因此按行和列访问时无 bank 冲突，也不需要额外空间。


参考 Swizzle 的实现，在 CUTLASS 中实现这个布局的方法是：
```c++
template <int BBits, int MBase, int SShift = BBits> struct SwizzleCircular {
  template <class Offset>
  CUTE_HOST_DEVICE constexpr static auto apply(Offset const &offset) {
    Offset num_cols = 1 << BBits;
    return offset / num_cols * num_cols +
           ((offset >> SShift) + (offset % num_cols)) % num_cols;
  }
  template <class Offset>
  CUTE_HOST_DEVICE constexpr auto operator()(Offset const &offset) const {
    return apply(offset);
  }
};
template <int B, int M, int S>
CUTE_HOST_DEVICE void print(SwizzleCircular<B, M, S> const &) {
  printf("SwCircular<%d,%d,%d>", B, M, S);
}
template <int B, int M, int S, class Shape, class Stride>
CUTE_HOST_DEVICE constexpr auto
composition(SwizzleCircular<B, M, S> const &sxor,
            Layout<Shape, Stride> const &layout) {
  return composition(sxor, _0{}, layout);
}

int main() {
  print_latex(composition(SwizzleCircular<3, 0>{},
                          make_layout(make_shape(8, 8), make_stride(8, 1))));
}
```

然而这个布局在计算时需要多次位运算分离行和列，计算量较大。尤其是加法，硬件实现时需要经过 ALU。因此 CUTLASS 中提出了基于异或的方法：
注意到前面限制列序号为 8 的公式是`(i + j) % 8`，这个公式实际上只要求在 i 和 j 其中一个不变的情况下，改变另一个可以遍历 0 到 7 范围内的整数，那么一个简单的化简是`i xor j`，这就是 CUTLASS 中 swizzle 布局的计算方法。这样的地址计算方法就是`offset ^ shiftr(offset & yyy_msk{}, msk_sft{})`，计算量很小。

![swizzle-cutlass](/assets/imgs/swizzle-cutlass.svg)

CUTLASS 的`swizzle<3,0,3>`布局，同样可以在按行和列访问时无 bank 冲突。


最后，顺便介绍一下 CUTLASS 中`Swizzle`的参数吧，官方文档里是这么写的：

```c++
// A generic Swizzle functor
/* 0bxxxxxxxxxxxxxxxYYYxxxxxxxZZZxxxx
 *                               ^--^ MBase is the number of least-sig bits to keep constant
 *                  ^-^       ^-^     BBits is the number of bits in the mask
 *                    ^---------^     SShift is the distance to shift the YYY mask
 *                                       (pos shifts YYY to the right, neg shifts YYY to the left)
 *
 * e.g. Given
 * 0bxxxxxxxxxxxxxxxxYYxxxxxxxxxZZxxx
 * the result is
 * 0bxxxxxxxxxxxxxxxxYYxxxxxxxxxAAxxx where AA = ZZ xor YY
 */
template <int BBits, int MBase, int SShift = BBits>
struct Swizzle{/*...*/};
```

三个参数的含义分别是：

 - 内存访问的基本单位为$$2^\textrm{MBits}$$个元素，这些元素作为一组，总是连续访问。具体一个“元素”代表什么取决于定义。
 - 矩阵中一行共有$$2^{\textrm{BBits}}$$个基本单位，即$$2^{\textrm{BBits}+\textrm{MBits}}$$个元素。
 - 访问模式的循环周期为$$2^{\textrm{SShift}}$$行，并且洗牌时$$2^{\textrm{SShift}}$$个基本单位作为一个整体。

用一个 TMA/`wgmma`中经常提到的布局， Swizzle 64B 来结尾吧：
64 字节 swizzle，每个元素大小为 2 字节
英伟达 GPU 中 Swizzle 的基本单位为 16 字节，图中一个元素为 2 字节，故`MBits`为 3。如果定义一个元素的含义为字节，则`Mbits`为 4。矩阵一行的长度为 64 字节，即 4 个基本单位，故`BBits`为 2。这个布局以两个基本单位为整体洗牌，并且每次访问 4 行有冲突，8 行才没有冲突，所以`SShift`为 3。 
至于其他的布局请参考 [CUDA 数据布局与 Tensor Core]({% post_url 2025-08-02-CUDA-Data-Layout %})。