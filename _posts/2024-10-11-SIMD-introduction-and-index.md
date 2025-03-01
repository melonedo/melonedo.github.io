---
layout: post
title: SIMD 简介及索引
date: 2024-10-11 18:23:00 +0800
tags: 
- 编程
render_with_liquid: false
---

## SSE/AVX

时至今日，SSE 已经成为 x86 系统中默认开启的功能。SSE4 中，定义了 128 位寄存器及对应的 SIMD 指令，这成为了后续 AVX 继续设计的基础。
AVX2 比 SSE 略新，一般需要指定架构才编译，不过实际上在大部分的 x86 CPU 中都启用。AVX 中定义了 256 位的寄存器，但是这些寄存器通常是作为两个同样的 128 位寄存器（lane）使用，必须使用`vperm`或是`vpack`等少数指令才能跨越 128 位边界，这是因为在不跨越 128 位边界的情况下，可以用两个 128 位的单元重叠地处理 256 位数据（double pumping），在同样的计算单元规模下，性能并不差，只是寄存器上下半间可能有一定延迟。而跨越边界的指令由 128 位单元处理可能非常缓慢，可能要几十个周期。
而 AVX512 是一组特性的集合，包括多个小的扩展，比如最基本的 AVX512F（Foundation）拓展了 512 位的寄存器及相关运算，并且把向量寄存器数量从 16 个拓展到 32 个。更加重要的是，AVX2 后续已经不再更新，[AVX512 实际上是 AVX3-AVX9](https://chipsandcheese.com/i/138977252/from-avx-to-avx-to-avx-to-avx)，如 AVX2 只定义了按偏移量读取数组的`vgather`系列指令，但没有定义对应的写入指令，这由 AVX512F 中的`vpscatter`系列指令补全。后续 AI 中常用的 16 位浮点，如[半精度](https://en.wikipedia.org/wiki/Half-precision_floating-point_format)、[bfloat16](https://en.wikipedia.org/wiki/Bfloat16_floating-point_format) 都包括在 AVX512 中。由于 AVX512 并不是一个单独的拓展，包括了十几个小的拓展，进行指令集判断时非常不便，因此英特尔在去年又推出了 [AVX10](https://cdrdv2-public.intel.com/784343/356368-intel-avx10-tech-paper.pdf)。AVX10 最主要的目标是简化 CPUID 的判断，即支持 AVX10 代表这个 CPU 支持大部分常用的 AVX512 拓展。AVX10 的另一个目的是支持 256 位的 AVX512，即支持 AVX512 中加入的各种新指令以及 32 个向量寄存器，只是大小限制在 256 位，否则 [不支持 AVX512 的 E 核只能使用 AVX2](https://www.anandtech.com/show/17047/the-intel-12th-gen-core-i912900k-review-hybrid-performance-brings-hybrid-complexity/2)。

虽然 AVX512 的 512 位运算推出很早，但是由于 AVX512 指令能耗过高，对应指令很长一段时间内都需要由 CPU [降频运行](https://lemire.me/blog/2018/09/07/avx-512-when-and-how-to-use-these-new-instructions/)，速度很可能不如 AVX2。直到最近的 Zen4、Zen5、Sunny Cove 才改变了现状，见：[Zen4's AVX512 Teardown](https://www.mersenneforum.org/node/21615?t=28102)、[Zen5's AVX512 Teardown + More...](http://www.numberworld.org/blogs/2024_8_7_zen5_avx512_teardown/)、[Ice Lake AVX-512 Downclocking](https://travisdowns.github.io/blog/2020/08/19/icl-avx512-freq.html)。

指令相关的资料，包括所用的[头文件](https://stackoverflow.com/a/11230437)和 C 函数名，可以在 [Intel® Intrinsics Guide](https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html) 查看。

- Cheats sheet: [x86 Intrinsics Cheat Sheet](https://db.in.tum.de/~finis/x86-intrin-cheatsheet-v2.1.pdf)
- Intel 官方文档：[Intel® 64 and IA-32 Architectures Optimization Reference Manual Volume 1](https://www.intel.com/content/www/us/en/content-details/814198/intel-64-and-ia-32-architectures-optimization-reference-manual-volume-1.html)。新的架构在第一卷，[第二卷](https://www.intel.com/content/www/us/en/content-details/821614/optimizing-earlier-generations-of-intel-64-and-ia-32-processor-architectures-throughput-and-latency.html)是比较旧的架构。Intel 的其他文档见：[Intel® 64 and IA-32 Architectures Software Developer Manuals](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)
- AMD 官方文档：[Software Optimization Guide for AMD EPYC™ 7003 Processors](https://www.amd.com/content/dam/amd/en/documents/epyc-technical-docs/software-optimization-guides/56665.zip) (Zen3)、[Software Optimization Guide for the AMD Zen4 Microarchitecture](https://www.amd.com/content/dam/amd/en/documents/epyc-technical-docs/software-optimization-guides/57647.zip)，[其他优化指南](https://www.amd.com/en/search/documentation/hub.html#sortCriteria=%40amd_release_date%20descending&f-amd_document_type=Software%20Optimization%20Guides)，[AMD64 Architecture Programmer’s Manual](https://www.amd.com/content/dam/amd/en/documents/processor-tech-docs/programmer-references/40332.pdf)。
- [WikiChip](https://en.wikichip.org/wiki/WikiChip)：各种架构的资料汇总，比维基专业很多。
- [Awesome-SIMD](https://github.com/awesome-simd/awesome-simd)。
- 一个讨论微架构的博客：[David Huang's Blog](https://blog.hjc.im/)。
- 一些 Hacker News 讨论：<https://news.ycombinator.com/item?id=41182395>、<https://news.ycombinator.com/item?id=30846226>、<https://news.ycombinator.com/item?id=30845228>。
- 一些教程：[Crunching Numbers with AVX and AVX2](https://www.codeproject.com/articles/874396/crunching-numbers-with-avx-and-avx)、[SIMD Parallelism - Algorithmica](https://en.algorithmica.org/hpc/simd/)、[TiFlash 面向编译器的自动向量化加速](https://tidb.net/blog/1886d9cd)。
- Dieshot：B 站的[万扯淡](https://space.bilibili.com/374034429)拍了非常非常多 dieshot，而 [LITTERTREE66](https://space.bilibili.com/381535123) 对其中不少 dieshot 做了注释。

## NEON

ARM 在 SIMD 方面则统一很多，只有一个 NEON 扩展，提供 128 位向量及浮点运算。NEON 的指令数量虽然不多，但设计相比 SSE/AVX 要整齐不少，比如在 swizzle 方面有包括`vldN/vstN/vrev/vext/vtrn/vzip/vuzip/vtbl/vtbx`等多个指令，参考：[Permutation - Neon instructions](https://developer.arm.com/documentation/102159/0400/Permutation---Neon-instructions)。

NEON 指令相关信息可以在 [Intrinsics - Arm Developer](https://developer.arm.com/architectures/instruction-sets/intrinsics/) 查看，注意里面的 [Helium](https://community.arm.com/arm-research/b/articles/posts/making-helium-why-not-just-add-neon) 是针对 M 系列 CPU 提出的 SIMD，又叫 MVE，而 SVE 目前在消费级产品上没有应用。NEON 所有的指令都位于`#include <arm_neon.h>`中。

## 宏定义

对于 GCC，使用下列命令获取架构相关的宏定义。
```
gcc -march=native -dM -E - < /dev/null | egrep "SSE|AVX|ARM" | sort
```

MSVC 的宏定义则直接在官网查看：[Microsoft-specific predefined macros](https://learn.microsoft.com/en-us/cpp/preprocessor/predefined-macros?view=msvc-170&redirectedfrom=MSDN#microsoft-specific-predefined-macros)。
