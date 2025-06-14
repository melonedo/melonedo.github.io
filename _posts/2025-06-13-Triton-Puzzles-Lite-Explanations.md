---
layout: post
title: Triton-Puzzles-Lite 解读
date: 2025-06-13 00:32:00 +0800
tags:
- 编程
- Triton
render_with_liquid: false
---

[Triton-Puzzles-Lite](https://github.com/SiriusNEO/Triton-Puzzles-Lite) 是目前 Triton 入门的最好的材料（没有之一）。目前网上有好几个讲解：

- 作者自己的介绍: [[MLSys 入门向] 做12道题，快速上手Triton！](https://zhuanlan.zhihu.com/p/5964285807)
- namoe 的讲解: [Triton魔法入门(1)--Triton-Puzzles-Lite](https://zhuanlan.zhihu.com/p/20539246076)
- 先进编译实验室的讲解：[【AI实操 · 优化篇】05 Triton算子开发](https://www.bilibili.com/video/BV193fFYkE7P)
- 以及这篇博客😋

# 前置知识

原版直接几个题目直接劈头盖脸就来了，非常不友好，我觉得还是得先简单介绍一些前置知识。由于目前没时间验证细节，都还是 TODO 状态。

## 安装运行

TODO，Triton-Puzzles-Lite感觉用起来有点神奇……需要特定的numpy和triton版本（和最新的已经隔了一个大版本了），得改不少东西。等有空再重新整理一下，还是先纸上谈兵一会吧

## 基本概念

和 CUDA 这种细粒度的 SIMT，或者是算子这种粗粒度不同，Triton 对于显卡上的运算提出了非常有趣的抽象。主要是要把一个计算的任务进行**划分**：

- 从任务层面，要把整个计算任务划分为多个基本不交互的子任务，称为**程序**（program）。这对应了显卡中 SIMT 编程范式所解决的最典型任务：过易并行（[Embarrassingly parallel](https://en.wikipedia.org/wiki/Embarrassingly_parallel)），也就是一个任务由成千上万个互相完全没有交互的子任务，例如将RGB图片转为灰度的过程中，每个像素的处理都是独立的，和别的像素完全不相关，可以随意地划分任务。
- 从数据层面，要把一个子任务所用到的数据划分为**瓦片**（tile），将一整块数据从内存中搬运到芯片上，计算后再写回内存。与 CPU 不同，显卡上的任务通常无法通过缓存金字塔的方法掩盖不同类型内存间的差异，并且通常需要处理远大于片上 SRAM 所能存储的数据量。因此，一个典型的显卡计算任务被建模为读取-计算-加载三个不同的阶段，用来清晰地说明数据在芯片间的流动，相比而言，CPU 上的计算任务不会划分这三个阶段，会在计算过程中随意地进行内存读写，寄希望于缓存能够掩盖内存访问延迟。

现实中的计算任务很多并不是过易并行的，并且也有非常大的内存带宽压力，因此上述两个层次的任务划分常常是并行算法设计的核心，必须由程序员亲自调教。而 Triton 认为两个任务适合于编译器完成：

- 数据布局：显卡中支持大规模的并行操作，但上述的并行并非如软件并行时一样任意，通常会受到对应的硬件端口限制，即产生寄存器/共享内存的 bank conflict 冲突。因此，数据按照哪种方法排布是非常讲究的，例如按行和按列读取矩阵是不同的，需要使用合适的方法处理，如在 CUTLASS 中通过各种各样的数据布局表示。
- 程序内部划分：显卡内部都是按照 SIMD 的方法进行计算的，具体哪个维度采用 SIMD 并行，哪个维度采用流水并行，可以由编译器根据计算的部分大致推断。

## 基本语法

以官方教程中的[向量加](https://triton-lang.org/main/getting-started/tutorials/01-vector-add.html#compute-kernel)为例，一个 Triton 核函数应该包含以下部分

```python
import triton
import triton.language as tl

# 这个函数会整个被作为 DSL 被 Triton 编译为对应显卡上的二进制程序，不会被 Python 执行
@triton.jit
def add_kernel(x_ptr, y_ptr, output_ptr, n_elements, # 一些运行时才知道的参数
               BLOCK_SIZE: tl.constexpr,  # 超参数，由于涉及张量大小，需要是常量
               ):
    # 子程序编号
    pid = tl.program_id(axis=0)
    # 计算偏移量和掩码，用于后续内存读写
    offsets = pid * BLOCK_SIZE + tl.arange(0, BLOCK_SIZE)
    mask = offsets < n_elements
    # 读取一块内存
    x = tl.load(x_ptr + offsets, mask)
    y = tl.load(y_ptr + offsets, mask)
    # 计算
    output = x + y
    # 写回内存
    tl.store(output_ptr + offsets, output, mask=mask)
```

形式上，Triton 核函数分为四个部分
1. 函数声明，包括若干个在运行时才能获取的参数，以及编译时已知的常量。
2. 子程序编号，使用`tl.program_id`获取，用来确定当前运行的子程序。
3. 张量内存读写，使用`tl.load`和`tl.store`进行，参数包括一组指针、对应掩码以及默认值。这方面有点类似 ARM/RISC-V/PTX 的汇编，而不像 CUDA/pytorch 中可以随意地用索引操作进行内存读写。
4. 具体对张量的计算。注意 Triton 中所有的张量大小都是在编译时已知的。

## 张量语法

在 Triton 中，各种整数、浮点数都可以用多维数组，也就是张量来表示。和 numpy/pytorch 中的张量类似，Triton 中的张量支持基本的[张量运算语义](https://triton-lang.org/main/python-api/triton-semantics.html)，包括：

- 广播：根据 numpy 规则，当两个张量运算时，如果对应的维度在一个操作数中是 1，则按照另一个操作数的维度进行广播操作。
- 类型提升：不同张量的类型运算时不会直接报错，而是会选定一个公共的类型储存结果。
- 索引操作：不知道支不支持通用的索引运算，但`None`这种维度上的操作肯定是支持的。
- 基本运算：加减乘除等逐元素的操作都支持，基本的归约操作也都支持。

TODO：确认索引操作支不支持通用的下标索引，按照 CUDA 来说是不支持的，这样的数组会被分配到分布在显存的 local memory 区域（没错， local 在 global 里）

## 万物起源：`tl.arange`

在 Triton 中，不基于现有张量，从头开始创建张量（`tl.tensor`）的方法十分有限：

- 通过`tl.arange`创建一个连续的张量序列（0, 1, 2, 3, ...）。
- 通过`tl.full/tl.zeros`创建一个所有元素相同的张量。

其中只有`tl.arange`可以创建一个元素各不相同的张量，用来计算索引等。并且，`tl.arange`还限制张量长度必须为2的幂。这反映了对应硬件上通常也以2的幂来进行资源分配，例如 CUDA 中线程束（warp）包括 32 个线程，一个线程块（block）通常包括 8 或 16 个线程束。（实际上，有了一组基，`tl.reshape`也可以用来构建任意尺寸的序列了，可能这样构建出来的序列性能并不好？）

TODO：确实没有别的方法了吗？