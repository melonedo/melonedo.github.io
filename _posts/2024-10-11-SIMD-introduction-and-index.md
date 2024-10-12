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
AVX2 比 SSE 略新，一般需要指定架构才编译，不过实际上在大部分的 x86 CPU 中都启用。AVX 中定义了 256 位的寄存器，但是这些寄存器通常是作为两个同样的 128 位寄存器（lane）使用，必须使用`vperm`或是`vpack`等少数指令才能跨越 128 位边界。
而 AVX512 是一组特性的集合，包括多个小的扩展，比如最基本的 AVX512F（Foundation）拓展了 512 位的寄存器，但使用上和上述的 256 位寄存器类似。更加重要的是，AVX2 指令集的空隙较多，如只定义了按偏移量读取数组的`vgather`系列指令，但没有定义对应的写入指令，这由 AVX512F 中的`vpscatter`系列指令补全。虽然 AVX512 的 512 位寄存器推出很早，但是由于 AVX512 指令能耗过高，对应指令很长一段时间内都需要由 CPU [降频运行](https://lemire.me/blog/2018/09/07/avx-512-when-and-how-to-use-these-new-instructions/)，速度很可能不如 AVX2。直到最近的 Zen4、Zen5 才改变了现状，见：[Zen4's AVX512 Teardown ](https://www.mersenneforum.org/node/21615?t=28102)、[Zen5's AVX512 Teardown + More...](http://www.numberworld.org/blogs/2024_8_7_zen5_avx512_teardown/)。

指令相关的资料，包括所用的[头文件](https://stackoverflow.com/a/11230437)和 C 函数名，可以在 [Intel® Intrinsics Guide](https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html) 查看。

- Cheats sheet: [x86 Intrinsics Cheat Sheet](https://db.in.tum.de/~finis/x86-intrin-cheatsheet-v2.1.pdf)
- Intel 官方文档：[Intel® 64 and IA-32 Architectures Optimization Reference Manual Volume 1](https://www.intel.com/content/www/us/en/content-details/814198/intel-64-and-ia-32-architectures-optimization-reference-manual-volume-1.html)。新的架构在第一卷，[第二卷](https://www.intel.com/content/www/us/en/content-details/821614/optimizing-earlier-generations-of-intel-64-and-ia-32-processor-architectures-throughput-and-latency.html)是比较旧的架构。Intel 的其他文档见：[Intel® 64 and IA-32 Architectures Software Developer Manuals](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)
- AMD 官方文档：[Software Optimization Guide for AMD EPYC™ 7003 Processors (Zen3)](https://www.amd.com/content/dam/amd/en/documents/epyc-technical-docs/software-optimization-guides/56665.zip)、[Software Optimization Guide for the AMD Zen4 Microarchitecture](https://www.amd.com/content/dam/amd/en/documents/epyc-technical-docs/software-optimization-guides/57647.zip)，[其他优化指南](https://www.amd.com/en/search/documentation/hub.html#sortCriteria=%40amd_release_date%20descending&f-amd_document_type=Software%20Optimization%20Guides)，[AMD64 Architecture Programmer’s Manual](https://www.amd.com/content/dam/amd/en/documents/processor-tech-docs/programmer-references/40332.pdf)。
- [Awesome-SIMD](https://github.com/awesome-simd/awesome-simd)。
- 一些 Hacker News 讨论：<https://news.ycombinator.com/item?id=41182395>、<https://news.ycombinator.com/item?id=30846226>、<https://news.ycombinator.com/item?id=30845228>。
- 一些教程：[Crunching Numbers with AVX and AVX2](https://www.codeproject.com/articles/874396/crunching-numbers-with-avx-and-avx)、[SIMD Parallelism - Algorithmica](https://en.algorithmica.org/hpc/simd/)。

## NEON

ARM 在 SIMD 方面则统一很多，只有一个 NEON 扩展，提供 128 位向量及浮点运算。NEON 的指令数量虽然不多，但设计相比 SSE/AVX 要整齐不少，比如在 swizzle 方面有包括`vldN/vstN/vrev/vext/vtrn/vzip/vuzip/vtbl/vtbx`等多个指令，参考：[Permutation - Neon instructions](https://developer.arm.com/documentation/102159/0400/Permutation---Neon-instructions)。

NEON 指令相关信息可以在 [Intrinsics - Arm Developer](https://developer.arm.com/architectures/instruction-sets/intrinsics/) 查看，注意里面的 [Helium](https://community.arm.com/arm-research/b/articles/posts/making-helium-why-not-just-add-neon) 是针对 M 系列 CPU 提出的 SIMD，又叫 MVE，而 SVE 目前在消费级产品上没有应用。NEON 所有的指令都位于`#include <arm_neon.h>`中。

## 宏定义

对于 GCC，使用下列命令获取架构相关的宏定义。
```
gcc -march=native -dM -E - < /dev/null | egrep "SSE|AVX|ARM" | sort
```

MSVC 的宏定义则直接在官网查看：[Microsoft-specific predefined macros](https://learn.microsoft.com/en-us/cpp/preprocessor/predefined-macros?view=msvc-170&redirectedfrom=MSDN#microsoft-specific-predefined-macros)。
