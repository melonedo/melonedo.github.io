---
layout: post
title: Triton 探秘：TTIR
date: 2025-08-31 20:24:00 +0800
tags:
- 编程
- CUDA
- Triton
render_with_liquid: false
---

关于 Triton 的编译过程，已经有不少的资料：

- [深度剖析 Triton编译器 MatMul优化（一）—— FMA](https://www.cnblogs.com/BobHuang/p/18810026)
- [Deep Dive into Triton Internals (Part 1)](https://www.kapilsharma.dev/posts/deep-dive-into-triton-internals/#ttir-triton-ir)
- [【AI实操 · 优化篇】01 Triton在PyTorch中的角色](https://www.bilibili.com/video/BV1ZoRPYQE2K/)

其中只有最后一个视频系列具体的研究了 Triton 的优化过程，不过也没有深入地探讨。但我认为研究 Triton 的优化过程非常重要，可以迁移到任何类似的 DSL 上，因此需要进行仔细地分析。

## 总览

Triton 的编译过程用到了多种层次的 IR：
- `@triton.jit`函数体使用 Python AST。
- Python AST 转换为 TTIR（Triton IR），进行通用优化，保证程序不会因为一些简单的写法变换而产生不同。
- TTIR 转换为 TTGIR（Triton GPU IR），进行 GPU 架构相关的优化。
- TTGIR 逐步转换为 LLVM IR，处理一些特殊的指令和结构。

其中除了 Python AST 以外，TTIR、TTGIR、LLVM IR 都使用 MLIR 的格式表示。不过，实际上 Triton 并没有用到很多 MLIR 的基建，例如所有特殊的 PTX 指令都是以内联汇编的形式表示的，和 CUDA 中用到层次完全相同，实际上现在的 MLIR 已经支持 NVGPU 后端（但非常孱弱，甚至 [bf16 都不支持](https://github.com/llvm/llvm-project/blob/1f2d461e26c4c7fdad922b1b62f2aced58844ce1/mlir/lib/Conversion/NVGPUToNVVM/NVGPUToNVVM.cpp#L315-L328)）。

万丈高楼平地起，今天先基于一个经典的 GEMM 代码，研究 Python AST 是如何变成 TTIR 的。


## 源代码

这是一个非常经典的 bf16 精度矩阵乘法，A 和 B 矩阵均不转置。精简起见只展示了核函数，完整的代码在 [matmul-with-dot-bf16.py](https://github.com/melonedo/Triton-blog-file/blob/main/matmul/kernel/matmul-with-dot-bf16.py)。

```python
@triton.jit
def matrix_multiplication_kernel(
    a_ptr, b_ptr, c_ptr,
    M, N, K,
    stride_am, stride_ak,
    stride_bk, stride_bn,
    stride_cm, stride_cn,
    BLOCK_SIZE_M: tl.constexpr, 
    BLOCK_SIZE_N: tl.constexpr,
    BLOCK_SIZE_K: tl.constexpr,
):
    a_ptr = a_ptr.to(tl.pointer_type(tl.bfloat16))
    b_ptr = b_ptr.to(tl.pointer_type(tl.bfloat16))
    c_ptr = c_ptr.to(tl.pointer_type(tl.float32))
    pid_n = tl.program_id(axis=0)
    pid_m = tl.program_id(axis=1)

    offs_m = pid_m * BLOCK_SIZE_M + tl.arange(0, BLOCK_SIZE_M)
    offs_n = pid_n * BLOCK_SIZE_N + tl.arange(0, BLOCK_SIZE_N)
    offs_k = tl.arange(0, BLOCK_SIZE_K)

    # 初始化A和B的指针
    a_ptrs = a_ptr + offs_m[:, None] * stride_am + offs_k[None, :] * stride_ak
    b_ptrs = b_ptr + offs_k[:, None] * stride_bk + offs_n[None, :] * stride_bn

    accumulator = tl.zeros((BLOCK_SIZE_M, BLOCK_SIZE_N), dtype=tl.float32)

    # 沿N维度按块累加
    for k in range(0, tl.cdiv(K, BLOCK_SIZE_K)):
        max_idx = K - k * BLOCK_SIZE_K
        # 加载A和B的块
        a = tl.load(a_ptrs + k * BLOCK_SIZE_K * stride_ak, mask=offs_k[None, :] < max_idx, other=0.0)
        b = tl.load(b_ptrs + k * BLOCK_SIZE_K * stride_bk, mask=offs_k[:, None] < max_idx, other=0.0)
        # 计算a @ b，累加到 accumulator
        accumulator += tl.dot(a, b)

    # 将结果写回C
    c_ptrs = c_ptr + offs_m[:, None] * stride_cm + offs_n[None, :] * stride_cn
    offs_cm = pid_m * BLOCK_SIZE_M + tl.arange(0, BLOCK_SIZE_M)
    offs_ck = pid_n * BLOCK_SIZE_N + tl.arange(0, BLOCK_SIZE_N)
    c_mask = (offs_cm[:, None] < M) & (offs_ck[None, :] < N)
    tl.store(c_ptrs, accumulator, mask=c_mask)
```

熟悉 GEMM 的读者应该已经对这个代码的结构倒背如流了，这个代码的内容非常地直接：

1. Triton 的代码都是 CTA（线程块）级别的，因此首先需要读取`tl.program_id`，实际上就是 CUDA 里面的 blockIdx。
2. 根据预设的分块大小，计算本 CTA 负责的 M 和 N 范围。
3. 沿着 K 维度迭代，加载 A 和 B 对应的块，使用`tl.dot`计算矩阵乘法。
4. 将结果写回 C。

在上述代码中，最为特色的是几个`tl`开头的函数，以及`[:, None]`类型的张量**索引**和相应的**广播**逻辑，这是 CUDA 中不具备的。后面在 TTIR 部分，我们会看到这几个表达对应的 TTIR 都并不简单。

## TTIR

使用下列命令，可以查看每个优化 pass 的结果。完整的 IR 非常冗长，因此文中只展示最相关的部分，相关的代码和完整的 IR 文件在 [melonedo/Triton-blog-file](https://github.com/melonedo/Triton-blog-file/tree/main/matmul/with_dot_bf16/MLIR) 仓库，有需要可以自行查看。

```shell
# pip install torch==2.8 triton==3.4 numpy
export MLIR_ENABLE_DUMP=1
export MLIR_DUMP_PATH=matmul/with_dot_bf16/MLIR/full.mlir
rm -r ~/.triton/cache/ # Optional, if nothing is dumped
python matmul/kernel/matmul-with-dot-bf16.py
python format-dump.py $MLIR_DUMP_PATH
```

其中`01-source.mlir`就是 Python AST 转换到未优化的 TTIR。源代码的对应关系非常重要，因此这部分的代码我做了注释，放在[01-source-commented.mlir](https://github.com/melonedo/Triton-blog-file/blob/main/matmul/with_dot_bf16/MLIR/01-source-commented.mlir)。这里讲解一下其中的关键部分。

### 溢出检查

由于 CUDA 中一个寄存器的大小只有 32 位，当数据特别大时，就有可能溢出。不过大部分情况下 32 位偏移量是足够的，因此 Triton 默认仍然是使用 32 位整数计算偏移量，因此用`@triton.jit(debug=True)`可以开启`tl.device_assert`，启用溢出检查，用 64 位整数检查计算结果。

```mlir
%2 = arith.extsi %1 : i32 to i64 loc(#loc3)
%3 = arith.extsi %c128_i32_0 : i32 to i64 loc(#loc3)
%4 = arith.muli %2, %3 : i64 loc(#loc3)
%c2147483647_i64 = arith.constant 2147483647 : i64 loc(#loc3)
%c-2147483648_i64 = arith.constant -2147483648 : i64 loc(#loc3)
%5 = arith.cmpi sle, %4, %c2147483647_i64 : i64 loc(#loc3)
%6 = arith.cmpi sge, %4, %c-2147483648_i64 : i64 loc(#loc3)
%7 = arith.andi %5, %6 : i1 loc(#loc3)
// 如果开启溢出检查，这里还有一行
// tt.assert %7, "int32 overflow detected for operation mul" : i1 loc(#loc3)
// 只有这行是实际计算的
%8 = arith.muli %1, %c128_i32_0 : i32 loc(#loc3)
```

### 索引计算

代码中，计算本 CTA 负责的 M 维度的表达式是：

```py
offs_m = pid_m * BLOCK_SIZE_M + tl.arange(0, BLOCK_SIZE_M)
```

对应的 TTIR 是：

```mlir
%c128_i32_0 = arith.constant 128 : i32 loc(#loc3)
// [检查溢出……]
%8 = arith.muli %1, %c128_i32_0 : i32 loc(#loc3)
// 对应 tl.arange(0, BLOCK_SIZE_M)
%9 = tt.make_range {end = 128 : i32, start = 0 : i32} : tensor<128xi32> loc(#loc4)
%10 = tt.splat %8 : i32 -> tensor<128xi32> loc(#loc5)
// [检查溢出……]
%17 = arith.addi %10, %9 : tensor<128xi32> loc(#loc5)
```

除了代码中已有的乘法和加法外，还插入了一个`tt.splat`，用于将标量类型转换为张量类型（`block_type`）。

### 指针计算

得到了索引后，还需要计算出指针。对应的源代码是：

```py
a_ptrs = (a_ptr + offs_m[:, None] * stride_am) + offs_k[None, :] * stride_ak
```

MLIR 是

```mlir
// offs_m[:, None] 对应 tt.expand_dims
%35 = tt.expand_dims %17 {axis = 1 : i32} : tensor<128xi32> -> tensor<128x1xi32> loc(#loc10)
%36 = tt.splat %arg6 : i32 -> tensor<128x1xi32> loc(#loc11)
// [检查溢出……]
%43 = arith.muli %35, %36 : tensor<128x1xi32> loc(#loc11)
// a_ptr -> tensor
%44 = tt.splat %arg0 : !tt.ptr<bf16> -> tensor<128x1x!tt.ptr<bf16>> loc(#loc12)
%45 = tt.addptr %44, %43 : tensor<128x1x!tt.ptr<bf16>>, tensor<128x1xi32> loc(#loc12)
%46 = tt.expand_dims %34 {axis = 0 : i32} : tensor<64xi32> -> tensor<1x64xi32> loc(#loc13)
%c1_i32_15 = arith.constant 1 : i32 loc(#loc14)
%cst_16 = arith.constant dense<1> : tensor<1x64xi32> loc(#loc14)
// [检查溢出……]
%53 = arith.muli %46, %cst_16 : tensor<1x64xi32> loc(#loc14)
// 处理广播
%54 = tt.broadcast %45 : tensor<128x1x!tt.ptr<bf16>> -> tensor<128x64x!tt.ptr<bf16>> loc(#loc15)
%55 = tt.broadcast %53 : tensor<1x64xi32> -> tensor<128x64xi32> loc(#loc15)
%56 = tt.addptr %54, %55 : tensor<128x64x!tt.ptr<bf16>>, tensor<128x64xi32> loc(#loc15)
```

注意上述计算的过程中，通过广播的方法，将两个一维的`tensor`组合成了一个二维的`tensor`。其中`[:, None]`表达式有对应的`tt.expand_dims`调整维度，并且还需要使用`tt.broadcast`处理广播。这两个操作都是 Python AST 中没有直接表达的，需要根据维度信息进行分析。

## TTIR 优化

TTIR 层次还没有考虑到 GPU 上任何的特殊性，因此只能做一些通用的死码消除、公共表达式消除、常量折叠、内联等。相关的优化定义在对应 backend 的`make_ttir`函数，分为 [NVIDIA](https://github.com/triton-lang/triton/blob/db379e8b34cfa3362e82f060ef0cd9f534bb13e7/third_party/nvidia/backend/compiler.py#L228-L243) 和 [AMD](https://github.com/triton-lang/triton/blob/db379e8b34cfa3362e82f060ef0cd9f534bb13e7/third_party/amd/backend/compiler.py#L186-L201) 两个版本。

例如代码最核心的内层循环部分，在优化前是：

```mlir
///////////////////////////////////////////////////////////////////////////
// accumulator = tl.zeros((BLOCK_SIZE_M, BLOCK_SIZE_N), dtype=tl.float32)
///////////////////////////////////////////////////////////////////////////
%79 = tt.call @"triton.language.standard.zeros____(0, 0)cconstexpr_128__(0, 1)cconstexpr_64__(1,)cconstexpr_fp32_"() : () -> tensor<128x64xf32> loc(#loc22)
///////////////////////////////////////////////////////////////////////////
// for k in range(0, tl.cdiv(K, BLOCK_SIZE_K)):
///////////////////////////////////////////////////////////////////////////
%80 = tt.call @"triton.language.standard.cdiv__i32__(1,)cconstexpr_64_"(%arg5) : (i32) -> i32 loc(#loc23)
%c0_i32 = arith.constant 0 : i32 loc(#loc24)
%c1_i32_32 = arith.constant 1 : i32 loc(#loc24)
%81 = arith.bitcast %c0_i32 : i32 to i32 loc(#loc24)
%82 = arith.bitcast %80 : i32 to i32 loc(#loc24)
%83 = arith.bitcast %c1_i32_32 : i32 to i32 loc(#loc24)
%84 = ub.poison : i32 loc(#loc24)
// 注意 scf 的写法，iter_args表示每个循环会改变的量
// %arg10 表示 accumulator，初始化为 %79（全零）
%85 = scf.for %arg9 = %81 to %82 step %83 iter_args(%arg10 = %79) -> (tensor<128x64xf32>)  : i32 {
    ///////////////////////////////////////////////////////////////////////////
    // max_idx = K - k * BLOCK_SIZE_K
    ///////////////////////////////////////////////////////////////////////////
    // BLOCK_SIZE_K=64
    %c64_i32_61 = arith.constant 64 : i32 loc(#loc25)
    // [检查溢出……]
    %155 = arith.muli %arg9, %c64_i32_61 : i32 loc(#loc25)
    // [检查溢出……]
    %162 = arith.subi %arg4, %155 : i32 loc(#loc26)
    %163 = tt.expand_dims %34 {axis = 0 : i32} : tensor<64xi32> -> tensor<1x64xi32> loc(#loc27)
    %164 = tt.splat %162 : i32 -> tensor<1x64xi32> loc(#loc28)
    %165 = arith.cmpi slt, %163, %164 : tensor<1x64xi32> loc(#loc28)

    ///////////////////////////////////////////////////////////////////////////
    // a = tl.load(a_ptrs + k * BLOCK_SIZE_K * stride_ak, mask=offs_k[None, :] < max_idx, other=0.0)
    ///////////////////////////////////////////////////////////////////////////
    %c64_i32_67 = arith.constant 64 : i32 loc(#loc29)
    // [检查溢出……]
    %172 = arith.muli %arg9, %c64_i32_67 : i32 loc(#loc29
    %c1_i32_71 = arith.constant 1 : i32 loc(#loc30)
    // [检查溢出……]
    %179 = arith.muli %172, %c1_i32_71 : i32 loc(#loc30)
    %180 = tt.splat %179 : i32 -> tensor<128x64xi32> loc(#loc31)
    %181 = tt.addptr %56, %180 : tensor<128x64x!tt.ptr<bf16>>, tensor<128x64xi32> loc(#loc31)
    %cst_74 = arith.constant 0.000000e+00 : f32 loc(#loc32)
    %182 = tt.broadcast %165 : tensor<1x64xi1> -> tensor<128x64xi1> loc(#loc32)
    %cst_75 = arith.constant dense<0.000000e+00> : tensor<128x64xf32> loc(#loc32)
    // 注意输入的是 f32，所以用 arith.truncf 转换一下
    %183 = arith.truncf %cst_75 : tensor<128x64xf32> to tensor<128x64xbf16> loc(#loc32)
    %184 = tt.load %181, %182, %183 : tensor<128x64x!tt.ptr<bf16>> loc(#loc32)

    ///////////////////////////////////////////////////////////////////////////
    // b = tl.load(b_ptrs + k * BLOCK_SIZE_K * stride_bk, mask=offs_k[:, None] < max_idx, other=0.0)
    ///////////////////////////////////////////////////////////////////////////
    // [和 a 差不多，省略……]
    %206 = tt.load %203, %204, %205 : tensor<64x64x!tt.ptr<bf16>> loc(#loc38)

    ///////////////////////////////////////////////////////////////////////////
    // accumulator += tl.dot(a, b)
    ///////////////////////////////////////////////////////////////////////////
    %cst_85 = arith.constant dense<0.000000e+00> : tensor<128x64xf32> loc(#loc39)
    %207 = tt.dot %184, %206, %cst_85, inputPrecision = tf32 : tensor<128x64xbf16> * tensor<64x64xbf16> -> tensor<128x64xf32> loc(#loc39)
    %208 = arith.addf %arg10, %207 : tensor<128x64xf32> loc(#loc40)
    scf.yield %208 : tensor<128x64xf32> loc(#loc41)
} loc(#loc24)
```

TTIR 优化的最后一个 pass 是`TritonLoopUnroll`，结果是：

```mlir
%cst_1 = arith.constant dense<0.000000e+00> : tensor<128x64xf32> loc(#loc1)
%c1_i32 = arith.constant 1 : i32 loc(#loc1)
%c0_i32 = arith.constant 0 : i32 loc(#loc1)
%28 = arith.addi %arg5, %c63_i32 : i32 loc(#loc43)
%29 = arith.divsi %28, %c64_i32 : i32 loc(#loc44)
%30 = scf.for %arg9 = %c0_i32 to %29 step %c1_i32 iter_args(%arg10 = %cst_1) -> (tensor<128x64xf32>)  : i32 {
    ///////////////////////////////////////////////////////////////////////////
    // max_idx = K - k * BLOCK_SIZE_K
    ///////////////////////////////////////////////////////////////////////////
    %45 = arith.muli %arg9, %c64_i32 : i32 loc(#loc24)
    %46 = arith.subi %arg5, %45 : i32 loc(#loc25)

    ///////////////////////////////////////////////////////////////////////////
    // a = tl.load(a_ptrs + k * BLOCK_SIZE_K * stride_ak, mask=offs_k[None, :] < max_idx, other=0.0)
    ///////////////////////////////////////////////////////////////////////////
    %47 = tt.splat %46 : i32 -> tensor<1x64xi32> loc(#loc26)
    %48 = arith.cmpi slt, %15, %47 : tensor<1x64xi32> loc(#loc26)
    %49 = tt.splat %45 : i32 -> tensor<128x64xi32> loc(#loc27)
    %50 = tt.addptr %18, %49 : tensor<128x64x!tt.ptr<bf16>>, tensor<128x64xi32> loc(#loc27)
    %51 = tt.broadcast %48 : tensor<1x64xi1> -> tensor<128x64xi1> loc(#loc28)
    %52 = tt.load %50, %51, %cst_0 : tensor<128x64x!tt.ptr<bf16>> loc(#loc28)

    ///////////////////////////////////////////////////////////////////////////
    // b = tl.load(b_ptrs + k * BLOCK_SIZE_K * stride_bk, mask=offs_k[:, None] < max_idx, other=0.0)
    ///////////////////////////////////////////////////////////////////////////
    %53 = tt.splat %46 : i32 -> tensor<64x1xi32> loc(#loc29)
    %54 = arith.cmpi slt, %19, %53 : tensor<64x1xi32> loc(#loc29)
    %55 = arith.muli %45, %arg7 : i32 loc(#loc30)
    %56 = tt.splat %55 : i32 -> tensor<64x64xi32> loc(#loc31)
    %57 = tt.addptr %27, %56 : tensor<64x64x!tt.ptr<bf16>>, tensor<64x64xi32> loc(#loc31)
    %58 = tt.broadcast %54 : tensor<64x1xi1> -> tensor<64x64xi1> loc(#loc32)
    %59 = tt.load %57, %58, %cst : tensor<64x64x!tt.ptr<bf16>> loc(#loc32)
    
    ///////////////////////////////////////////////////////////////////////////
    // accumulator += tl.dot(a, b)
    ///////////////////////////////////////////////////////////////////////////
    %60 = tt.dot %52, %59, %arg10, inputPrecision = tf32 : tensor<128x64xbf16> * tensor<64x64xbf16> -> tensor<128x64xf32> loc(#loc33)
    scf.yield %60 : tensor<128x64xf32> loc(#loc34)
} loc(#loc23)
```

主要的变化是：

- 没有用到的溢出检测相关代码消除了。
- 用到的`tl.zeros`和`tl.cdiv`内联到了核函数里。
- `accumulator += tl.dot(a, b)`优化成了`tl.dot(a, b, accumulator)`。

可以看到 TTIR 上能做的基本上程序稍微调整一下结构也能做，主要是防止程序写错一点就差别巨大。再下一个 pass 就要将 TTIR 转换为 TTGIR 了，下一篇讲。
