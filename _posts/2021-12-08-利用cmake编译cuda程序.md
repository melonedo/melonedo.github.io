---
layout: post
title: 利用CMake编译CUDA程序
date: 2021-12-08 20:03:42 +0800
tags: 
- 编程
- CMake
---

代码：[melonedo/cmake-lecture/CUDA](https://github.com/melonedo/cmake-lecture/tree/main/cuda)

CUDA教程中的编译方法通常是直接调用`nvcc`（windows需在x64 Native Tools Command Prompt中运行）或者在vs中CUDA集成，前者不太方便自动控制，后者比较麻烦，我尝试使用cmake生成。

CUDA编译的东西很多，这里只编译一个独立可执行文件。

## VS集成

[Building issues [no CUDA toolset found] · Issue #103 · mitsuba-renderer/mitsuba2 · GitHub](https://github.com/mitsuba-renderer/mitsuba2/issues/103#issuecomment-618378963)

VS2019需要CUDA 10.1以上，另外在我的电脑上出现了能找到`nvcc`却找不到CUDA的问题，解决方案如上。

## CUDA样例

来自[An Even Easier Introduction to CUDA \| NVIDIA Developer Blog](https://developer.nvidia.com/blog/even-easier-introduction-cuda/)。命名为**main.cu**。

```c++
#include <math.h>
#include <iostream>

// Kernel function to add the elements of two arrays
__global__ void add(int n, float *x, float *y) {
  for (int i = 0; i < n; i++) y[i] = x[i] + y[i];
}

int main(void) {
  int N = 1 << 20;
  float *x, *y;

  // Allocate Unified Memory – accessible from CPU or GPU
  cudaMallocManaged(&x, N * sizeof(float));
  cudaMallocManaged(&y, N * sizeof(float));

  // initialize x and y arrays on the host
  for (int i = 0; i < N; i++) {
    x[i] = 1.0f;
    y[i] = 2.0f;
  }

  // Run kernel on 1M elements on the GPU
  add<<<1, 1>>>(N, x, y);

  // Wait for GPU to finish before accessing on host
  cudaDeviceSynchronize();

  // Check for errors (all values should be 3.0f)
  float maxError = 0.0f;
  for (int i = 0; i < N; i++) maxError = fmax(maxError, fabs(y[i] - 3.0f));
  std::cout << "Max error: " << maxError << std::endl;

  // Free memory
  cudaFree(x);
  cudaFree(y);

  return 0;
}
```

## CMake模板

来自[Building Cross-Platform CUDA Applications with CMake \| NVIDIA Developer Blog](https://developer.nvidia.com/blog/building-cuda-applications-cmake/)。需要编译动态库的话见原文。

```cmake
cmake_minimum_required(VERSION 3.12 FATAL_ERROR)
project(cmake_and_cuda LANGUAGES CXX CUDA)

# INTERFACE库enable_utf8用于在msvc中启用utf-8
add_library(enable_utf8 INTERFACE)
if(MSVC)
    # cmake的generator expression可以根据语言选择参数，编译c++时使用前者，CUDA使用后者
    target_compile_options(enable_utf8 INTERFACE
        $<$<BUILD_INTERFACE:$<COMPILE_LANGUAGE:CXX>>: /utf-8 >
        $<$<BUILD_INTERFACE:$<COMPILE_LANGUAGE:CUDA>>: -Xcompiler=/utf-8 >
    )
endif()

add_executable(test main.cu)

# 指定显卡架构，可传入列表指定多个架构。CMake 3.23以上可以用all或者all_major
set_target_properties(cuda-test PROPERTIES CUDA_ARCHITECTURES 75)

# 连接到INTERFACE库enable_utf8即可启用utf-8
target_link_libraries(test PRIVATE enable_utf8)
# 注：INTERFACE表示被连接到的目标使用，PRIVATE表示库自己使用，PUBLIC为前两者并集，即自己和被连接到的项目都可以。
```

基本上就是语言里加个CUDA就解决了，其他的部分是在解决编码的问题。如果用VS，需要借助`nvcc`向msvc传选项`/utf-8`来避免编码问题，添加的方法来自[passing flags to nvcc via CMake - #2，来自 qiyupei - CUDA Programming and Performance - NVIDIA Developer Forums](https://forums.developer.nvidia.com/t/passing-flags-to-nvcc-via-cmake/75768/2)。（居然是OSPP项目导师，巧了）

显卡架构分为虚架构和实架构，虚架构即使用的软件API版本，实架构代表硬件API版本，当虚版本低于当前电脑的硬件时CUDA将JIT生成需要的代码。可以参考[StackOverflow问题differences between virtual and real architecture of cuda](https://stackoverflow.com/questions/14779523/differences-between-virtual-and-real-architecture-of-cuda)、[NVCC手册关于GPU编译的部分](https://docs.nvidia.com/cuda/cuda-compiler-driver-nvcc/index.html#gpu-compilation)。关于自己的显卡是什么型号的可以参考[维基：CUDA#GPUs supported](https://en.wikipedia.org/wiki/CUDA#GPUs_supported)。

另有[`FindCUDAToolkit`](https://cmake.org/cmake/help/latest/module/FindCUDAToolkit.html#findcudatoolkit)或者`FindCUDA`方法，~~不太清楚~~ 在新版的CMake更新支持CUDA语言后不建议使用。

### [`FindCUDAToolkit`](https://cmake.org/cmake/help/latest/module/FindCUDAToolkit.html#findcudatoolkit)

上述配置用于编译 *.cu 文件，而如果要在 *.cpp 文件中使用，需要链接到 cudart 或是其他的库，如nvjpeg：

```cmake
find_package(CUDAToolkit REQUIRED)
target_link_libraries(main PRIVATE CUDA::cudart CUDA::nvjpeg)
```

具体要链接什么库参考 FindCUDAToolkit 文档。如果 nvcc 没有装在系统目录，可以使用CUDAToolkit_ROOT

### [`FindCUDA`](https://cmake.org/cmake/help/latest/module/FindCUDA.html)

OpenCV 仍然使用 FindCUDA 设置 CUDA 相关内容。要注意的是，FindCUDA 使用`-DCUDA_TOOLKIT_ROOT_DIR=xxx`设置自定义路径，而 FindCUDAToolkit 虽然在文档里说使用`-DCUDAToolkit_ROOT=xxx`设置，实际上需要使用`-DCMAKE_CUDA_COMPILER=xxx/bin/nvcc`，并且两者的默认搜索规则不同。两者如果不同时可能会产生如 stack smashing detected 的错误，说明编译和链接时使用的库不同。可以在 CMakeCache.txt 里搜索 libcudart 相关内容确定搜索到的是同一个库。

## 运行

可执行文件可以直接运行，如果需要profile可以`nvprof 文件名`。
