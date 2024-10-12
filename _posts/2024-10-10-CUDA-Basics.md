---
layout: post
title: CUDA 基础
date: 2024-10-10 22:47:00 +0800
tags: 
- 编程
---

CUDA 是目前大规模并行加速唯一的通用平台，要想大规模地加速如矩阵运算、神经网络这样的以并行计算为主的任务，目前仍然基本只能选择购买英伟达的 GPU，并使用 CUDA 进行编程。为此，这里介绍一些使用的要点。

# 参考资料
- [CUDA Toolkit Documentation](https://docs.nvidia.com/cuda/)：CUDA 相关文档的总目录。
- [CUDA Runtime API](https://docs.nvidia.com/cuda/cuda-runtime-api/index.html)：CUDA 运行时（cuda_runtime.h）中函数的文档。
- [CUDA C++ Programming Guide](https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html)：CUDA（.cu）语言及编程环境相关的官方参考，内容非常全面。
- [CUDA C++ Best Practices Guide](https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/index.html)：CUDA 编程的最佳实践，主要是纲领型的内容，细节还是参考上面这个。
- [Nsight Systems](https://docs.nvidia.com/nsight-systems/index.html) 和 [Nsight Compute](https://docs.nvidia.com/nsight-compute/index.html)：CUDA profiling 的工具，前者是针对整个程序，可以了解包括 CPU 占用 GPU 计算占用率、内存传输占用率等，后者针对单个 CUDA 核，分析具体的信息。以前很多教程用的是 [Visual Profiler 和 nvprof](https://docs.nvidia.com/cuda/profiler-users-guide/index.html)，现在似乎官方已经不推荐。

# 命名

CUDA 文档阅读中，一般不会直呼产品的名称，而是只提架构名以及 Compute Capibility，相当于版本代号及版本号。在 [CUDA GPUs - Compute Capability](https://developer.nvidia.com/cuda-gpus) 可以查到各个型号对应的称呼。近几年的消费级产品的对应关系大致如下：

|       产品 | 架构         | Compute Capability |
|-----------:|--------------|--------------------|
| 16系、20系 | Turing       | 7.5                |
|       30系 | Ampere       | 8.6                |
|       40系 | Ada Lovelace | 8.9                |
|     Orin系 | Ampere+Tegra | 8.7                |

# 软件

要开发一个 CUDA 的程序，需要包括三个部分的软件辅助：

- 开发环境：用于编译，在<https://developer.nvidia.com/cuda-downloads>获取安装方式。这是个动态页面，耐心等待。这里要注意的是，CUDA 开发环境默认是安装在 /usr/local/cuda-xx.x 中，即使安装了多个，也可以通过切换路径来选择。
- 用户态驱动：运行时链接的动态库，一般随驱动安装。
- 内核态驱动：底层的驱动，除了 Tegra 系列只能随系统升级以外，基本上是上面那个网址里的介绍安装一个包即可。

可以看到，上述几个库间存在依赖关系，由于硬件驱动一般不好升级，因此运行环境兼容不太旧的硬件驱动。而开发环境为了利用最新的功能，要求匹配新的运行环境。因此，一般尽可能用新的运行环境，而开发环境则在不影响功能的情况下尽可能用旧的。具体细节见 [CUDA Compatibility Developer’s Guide](https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/index.html#cuda-compatibility-developer-s-guide)

另外，CUDA 配套的示例程序，如实用的[`deviceQuery`](https://github.com/NVIDIA/cuda-samples/tree/master/Samples/1_Utilities/deviceQuery)程序，已经不再打包到开发环境中，需要前往 <https://github.com/NVIDIA/cuda-samples> 自行克隆并编译运行。

# 内存

对于独显而言，CPU 和显卡和各自的内存相连，互相只能通过 PCIE 总线通信。一般来说，现在的 Zen 4 或者是 14 代 CPU 加上高频 DDR5 内存，访问速度能达到 70GB/s，而显存的速度更是大约 1TB/s，然而 PCIE 4.0 x 16 的速度仅有 32GB/s，因此独显设备和 CPU 间数据传输是开发中关键的问题之一。

```
内存 <----> CPU <--PCIE--> 独显 <----> 显存
```

因此，CUDA 编程中，CPU 端的内存常分为以下几类：

- 普通内存（Pageable Memory）：没有和 CUDA 驱动申请过的内存，显卡完全无法访问。使用`cudaMemcpy`时，会首先把普通内存复制到下面的共享内存中。
- 共享内存（Pagelocked/Pinned/Mapped/Zero-copy Memory）：不会在硬盘里，因此可以直接做 DMA，无需 CPU 处理，因此执行`cudaMemcpy`的速度非常快。普通内存经过`cudaHostRegister`注册会成为共享内存，通过`cudaMallocHost`等分配内存时自动执行这一步。。这几个英文名的含义实际上不同，但是在比较新的版本中一注册就自动获取上述所有属性，区别是显卡侧使用的方法。
- 统一内存（Unified/Managed Memory）：可能在显存里，也可能在内存里。通过`cudaMallocManaged`或`__managed__`关键字分配。

另外，CPU 侧内存根据缓存策略，还细分为只读、CPU 不缓存、和所有 CUDA 语境共享等属性，这在简单的程序中通常不涉及。

GPU 端的显存则分为：

- 普通显存：和普通的 CPU 内存类似，只能由显卡访问，通过`cudaMalloc`分配。完全是显卡自己的内存，因此可以走显卡自己的内存控制器和缓存，速度非常快。CPU 侧无法直接访问普通的显存，只能通过`cudaMemcpy`复制。
- 映射内存：前面的共享内存在比较新的版本后自动映射到显卡中，通过`cudaHostGetDevicePointer`获取映射的地址。支持统一寻址（Unified Virtual Addressing）的系统里一般和 CPU 中的指针相同。本质上是 CPU 管理的内存，在显卡需要使用时，再按需请求需要的内存，可以减少一次完整的`cudaMemcpy`延迟。独显可以直接访问映射内存，但经过 CPU 转手，速度非常慢，一般还是复制到显存再用。
- 统一内存：独显的显存和 CPU 的内存并不在一起，因此统一内存在使用时，会一整块一起**迁移**到使用的设备中。例如，显卡在第一次访问统一内存后，整块内存都会迁移到显卡中，方便后续的访问。而 CPU 访问了上述内存后，整块统一内存又会迁移到内存中，方便 CPU 访问。由于上述的同步过程实际上涉及大量的软硬件支持，实际上使用时可能并不迁移，而只是固定在内存或是显存中，按需传输。

对于 Tegra 系列，显存和内存实际上是同一块物理内存，因此三种内存的速度几乎一样，区别只是 CPU 和 GPU 的缓存策略，这时可以把片上缓存当成一个小的内存或是显存来理解。不过由于内存控制器及缓存的原因，显存的速度明显更快。参考：[CUDA for Tegra: Table 1. Characteristics of Different Memory Types in a Tegra System](https://docs.nvidia.com/cuda/cuda-for-tegra-appnote/index.html#id5)。


# 计算

使用 CUDA 的意义当然是为了计算，而 CUDA 的计算模式实际上非常地独特。

- 一个显卡具有多个流多处理器（Streaming Multiprocessor），一个流多处理器可以同时运行成千上万个核函数实例。和 CPU 上类似，每个实例称为线程（Thread），一个流多处理器可能同时重叠执行多组线程，提高总流量。
- 但这么多核函数并不是各自独立执行的，而是分为以**32**个实例为一组的 Warp，每个 Warp 同时执行，共享 PC 指针，即使是有分支，也是不同的分支选择性执行。类似于使用 SIMD 将上述的核函数在 CPU 上并行化。
- 而流多处理器并不直接调度每个 Warp，而是把 Warp 包装成一个个 Block，在计算资源足够的情况下，重叠地执行多个 Block。
- 显卡中有几十个流多处理器，因此可以迅速地执行由多个 Block 组成的任务。
- 复杂的任务中，还需要处理缓存、寄存器、共享内存等的限制。

## 启动核函数

由于核函数的上述调度限制，在启动时，需要指定四个参数：

- `gridDim`：启动 Block 的数量，可以根据需要组成成`dim3`。
- `blockDim`：每个 Block 中线程的数量，根据每个 Block 所需要的计算资源确定，由若干 Warp 组成，所以一般是 32 的倍数，通常取 256 或者 128 即可。
- 额外动态申请的共享内存大小，不需要时写 0。
- 流，用于声明依赖关系。

因此，启动一个核函数的完整语法是

```cuda
kernel<<<blocksPerGrid, threadsPerBlock, extra_shmem, stream>>>(args...);
```

## 核函数

在计算时，需要参考上面的编程指南，编写核函数（Kernel），并调度核函数。以[向量加法样例](https://github.com/NVIDIA/cuda-samples/blob/master/Samples/0_Introduction/vectorAdd/vectorAdd.cu)为例，说明核函数的基本编写方法：

```cuda
__global__ void vectorAdd(const float *A, const float *B, float *C, int numElements) {
  int i = blockDim.x * blockIdx.x + threadIdx.x;

  if (i < numElements) {
    C[i] = A[i] + B[i] + 0.0f;
  }
}
```

- 使用 CUDA 扩展的 C++ 进行编写，编译的方法参考以前的博客：[利用cmake编译cuda程序]({% post_url 2021-12-08-利用cmake编译cuda程序 %})。
- 可以在显卡中运行的函数所使用的指令集等和 CPU 上的函数完全不同，因此核函数用`__global__`标注，普通的显卡中的函数用`__device__`标注。
- 和 CPU 中的函数不同，不同的核函数调用并不通过参数区分，而是以特殊变量`threadIdx`、`blockIdx`、`blockDim`区分，一般按照`i = blockDim.x * blockIdx.x + threadIdx.x`形式组合。另外还有一个`gridDim`，一般没什么用。
- 获取了`i`之后，就如同`for`循环的一次迭代，读取显存，计算结果，写回显存。注意一组核函数的参数完全一致，没有机会返回单独的值，因此只能通过读写内存的方法与外界交互。
- 如果是复杂的程序，可能还有多种同步的任务。

## 流

在 CUDA 中，数据依赖关系最常用的表示方式是流（Stream），在同一个流中，各个核函数、内存复制等操作都是依次进行的，不需要显式同步，类似于一个 CPU 线程。而不同的流之间有可能互相重叠，GPU 同时处理多个流的数据，从而实现类似于 CPU 超线程的并行处理，同时做多组计算和数据传输，充分利用计算资源。
不过，根据 [Steve Rennich 的演讲](https://developer.download.nvidia.com/CUDA/training/StreamsAndConcurrencyWebinar.pdf)，流间任务的调度实际上处理逻辑比较简单，并不非常智能：

- GPU 中有多个全局的队列，包括两个方向各字的复制任务队列，以及一个计算队列
- 每个队列中的任务**依次**启动运行。
- 如果队列中前面的任务已经开始，还未结束，但有计算资源而且当前任务的依赖已经满足，则继续启动任务。
- 如果计算队列结束后，有新的计算任务加入，即使依赖满足，也**不启动**新的内存复制任务。

其中依次启动是比较重要的一点，即如果两个流都运行三个计算任务`A B C`，并且计算资源充足时

- 当队列是`A1 B1 C1 A2 B2 C2`，GPU 中运行的顺序是`A1 B1 C1A2 B2 C2`
- 而当队列是`A1 A2 B1 B2 C1 C2`时，GPU 中运行的顺序是`A1A2 B1B2 C1C2`。

而最后一条意味着，如果同时启动了多个计算任务，会等到其中所有的计算任务都结束再开始内存传输，如上述C改为传输任务，

- 在计算队列是`A1 A2 B1 B2`，复制队列是`C1 C2`时，运行顺序是`A1A2 B1B2 C1 C2`，即 B1 运行结束后，C1 并没有马上开始，而是等到 B2 也结束才启动。

## 内存模型

和 CPU 一样，在一个 Warp 中，显存（在核函数的语境中称为 Global Memory）和共享内存并不以字节为单位，而是会组织成至少 **32** 个字节的内存事务，同时访问连续并且对齐的 32 个字节。因此要求显存中的数据应该按照至少 32 个字节对齐（OpenCV 甚至按照 512 字节对齐，可能是根据材质（Texture）的要求做的）。即时只访问了 32 个字节中的一部分，也需要读取完整的 32 个字节。当然，如果一个 Warp 中的所有 32 个线程访问连续的 32 个字节，这个要求是自动满足的。

而共享内存是一个 Block 中共享的资源，相比显存更加灵活，组成成 32-way associative 的缓存的形式。共享内存根据低位分成 32 组（Bank），一次共享内存操作中，分别访问 32 组中的**一个**位置（4字节），不要求连续。这意味着，如果一个 Warp 向共享内存的操作如果**去掉重复**的部分，分别属于不同的 Bank，则操作可以一次完成。如果有 n 个操作属于同一个 Bank，即产生 Bank Conflict，则需要分成多次访问。

更多细节参考：[CUDA C++ Programming Guide: 5.3.2. Device Memory Accesses](https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html#device-memory-accesses)和 [Compute Capability 5.x 的共享内存模型](https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html#shared-memory-5-x)。

## 调试

如果显存访问越界，可以使用`compute-sanitizer`运行程序，会在出错时提供说明。更细致的调试由`cudagdb`提供，对应的 [VSCode 插件](https://docs.nvidia.com/nsight-visual-studio-code-edition/cuda-debugger/index.html)。
