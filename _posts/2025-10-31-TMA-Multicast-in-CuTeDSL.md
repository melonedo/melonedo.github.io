---
layout: post
title: CuTe DSL 中的 TMA 多播接口
date: 2025-10-31 10:40:00 +0800
tags:
- 编程
- CUDA
render_with_liquid: false
---

最近在看 CuTe DSL 中 [hopper/dense_gemm.py](https://github.com/NVIDIA/cutlass/blob/v4.2.1/examples/python/CuTeDSL/hopper/dense_gemm.py) 示例，顺便学习一下 Hopper 下 CuTe DSL 中的 TMA 接口涉及，主要是分析多播的相关处理。

# 什么是 TMA

对于一个需要编写算子的人来说，TMA 是众多内存搬运的方式之一，用于从全局内存（GMEM）和共享内存（SMEM）间搬运数据。借助于 CuTe 的 [Tiled Copy](https://zhuanlan.zhihu.com/p/1930389542784964333) 抽象，算子工程师在编写代码时，只需要知道以下内容：

- 和所有的 tiled copy 一样，使用`cute.copy`命令即可将一整块内存进行复制，而无需考虑该命令内部具体是否再细分为多个更小的参数。和其他的 tiled copy 相同，CuTe 无法再提供更多的抽象，该函数前后需要维护大量其他的状态。
- 给定特定的 GMEM 和 SMEM 布局，使用 CuTe DSL 中的特定接口，即可产生用于 kernel 使用的参数。不过，其中的 TMA 描述符计算相当复杂，通常会在 host 端完成。
- TMA 进行 GMEM->SMEM 搬运时，需要使用 [mbarrier](https://zhuanlan.zhihu.com/p/1956123349592805422) 进行同步，对应的状态管理相当复杂。
- TMA 由一个线程负责发射，因此不需要考虑 tiled copy 中复杂的 TV-layout 以及`partition_S/D`。
- 取而代之的，是 TMA 可以进行 SM 间的多播（multicast），多个 SM 上的多个 CTA，类似于多个线程，但使用的接口不同。

篇幅所限，本文无法介绍 TMA 的原理，不太熟悉的读者可以参考[渣导](https://mp.weixin.qq.com/s/9PKZ9zks1Mx5qaNFqIX3XA)或者 [COLFAX](https://research.colfax-intl.com/tutorial-hopper-tma/)。

# TMA 流程

首先先看看一个基本的 TMA 程序在 CuTe DSL 中包含哪些部分。

1. 确定 GMEM tensor 和 SMEM layout。显然，这两者是有一些内存对齐、swizzle 模式等的约束的。
2. 使用`cute.nvgpu.cpasync.make_tiled_tma_atom`，确定所用到的 copy atom，顺便转换一下 GMEM tensor 的布局为 [scaled basis](https://docs.nvidia.com/cutlass/media/docs/cpp/cute/0z_tma_tensors.html)。这一步在 host 完成，应该是顺便创建了 TMA 描述符。
3. 划分出对应的 GMEM tensor，并分配好对应的 SMEM tensor。
4. 使用`cute.nvgpu.cpasync.tma_partition`进行一次布局变换，类比`ThrCopy.partition_S/D`。
5. 如果是 GMEM-SMEM，需要初始化对应的 mbarrier，需要计算 TMA 传输总大小，以及打算让多少个线程到达（arrive）。
6. 调用`cute.copy`执行，然后使用 mbarrier 或者 Bulk async-group 方法等待传输完成。

另外，和所有的内存操作一样，还需要考虑内存一致性要求，插入相关的 fence。

下面以 [hopper/dense_gemm.py](https://github.com/NVIDIA/cutlass/blob/v4.2.1/examples/python/CuTeDSL/hopper/dense_gemm.py) 示例中的 TMA 操作为参考，具体探究一下相关的接口。为了简明起见，只写 A 矩阵相关操作，B 矩阵的操作省略，在需要同时考虑 A 和 B 的地方不省略 B 相关内容。

## Host 侧准备工作

第 1 步 和 TMA 没有直接关系，省略。第 2 步在`_make_tma_atoms_and_tensors`函数完成：

```python
class HopperWgmmaGemmKernel:
    @cute.jit
    def __call__(...):
        # ...
        tma_atom_a, tma_tensor_a = self._make_tma_atoms_and_tensors(
            a,                         # GMEM tensor
            self.a_smem_layout_staged, # SMEM layout
            (self.tile_shape_mnk[0], self.tile_shape_mnk[2]), # e.g. 128x256
            self.cluster_shape_mn[1],  # e.g. 2
        )
        # ...
        self.kernel(
            tma_atom_a,
            tma_tensor_a,
            self.cta_layout_mnk,
            self.a_smem_layout_staged,
            # ...
        ).launch(...)

    @staticmethod
    def _make_tma_atoms_and_tensors(
        tensor: cute.Tensor,
        smem_layout_staged: cute.ComposedLayout,
        smem_tile: tuple[int, int],
        mcast_dim: int,
    ) -> tuple[cute.CopyAtom, cute.Tensor]:
        op = (
            cute.nvgpu.cpasync.CopyBulkTensorTileG2SOp()
            if mcast_dim == 1
            else cute.nvgpu.cpasync.CopyBulkTensorTileG2SMulticastOp()
        )

        # (M, K, NumStage) -> (M, K)
        smem_layout = cute.slice_(smem_layout_staged, (None, None, 0))
        tma_atom, tma_tensor = cute.nvgpu.cpasync.make_tiled_tma_atom(
            op,
            tensor,
            smem_layout,
            smem_tile,
            num_multicast=mcast_dim,
        )
        return tma_atom, tma_tensor
```

可以看到这个过程抽象得很好，除了多播的情况下需要指定一个特殊的 op 以外，只调用了一个`cute.nvgpu.cpasync.make_tiled_tma_atom`函数。

## Device 侧准备工作

第 3 步和所有的 kernel 一样，进入 kernel 都需要读取 thread idx 和 block idx，并划分 GMEM tensor。其中比较特别的是使用了 TMA 时，还需要预取 TMA 描述符。

由于我们考虑的是更加通用的涉及多播的 TMA，还需要获取 cta rank，实际上就是当前 CTA 在 CTA cluster 中的序号。CTA cluster 是在 thread/CTA 之上的又一个通信层次，基于 TMA 多播功能，可以利用 CTA cluster 进一步减少 L2 的访存。COLFAX 的 [CUTLASS Tutorial: GEMM with Thread Block Clusters on NVIDIA® Blackwell GPUs](https://research.colfax-intl.com/cutlass-tutorial-gemm-with-thread-block-clusters-on-nvidia-blackwell-gpus/) 一文对此有深入的介绍。


```python
class HopperWgmmaGemmKernel:
    @cute.kernel
    def kernel(
        self,
        tma_atom_a: cute.CopyAtom,
        mA_mkl: cute.Tensor,
        cta_layout_mnk: cute.Layout,
        # ...
    ):
        warp_idx = cute.arch.make_warp_uniform(cute.arch.warp_idx())

        # 预取 TMA 描述符
        if warp_idx == 0:
            cute.nvgpu.cpasync.prefetch_descriptor(tma_atom_a)

        tidx, _, _ = cute.arch.thread_idx()
        bidx, bidy, bidz = cute.arch.block_idx()
        cta_rank_in_cluster = cute.arch.make_warp_uniform(
            cute.arch.block_idx_in_cluster()
        )
        cluster_coord_mnk = cta_layout_mnk.get_flat_coord(cta_rank_in_cluster)

        # 划分 GMEM tensor
        # tile_coord_mnkl 是一个包括了 thread block swizzle 的复杂坐标
        gA_mkl = cute.local_tile(
            mA_mkl, self.tile_shape_mnk, tile_coord_mnkl, proj=(1, None, 1)
        )
        # 分配 SMEM tensor
        storage = smem.allocate(self.shared_storage)
        sA = storage.sA.get_tensor(
            a_smem_layout_staged.outer, swizzle=a_smem_layout_staged.inner
        )
```

## Tensor 布局变换

第 4 步是进行一次布局的变换。特别的地方是 TMA 使用的是`tma_partition`而不是经典的`partition_S/D`。`tma_partition`看起来只作用于第一组维度，需要使用一次`group_mode`处理。也是注意到 TMA 是一个 CTA 级别的函数，这里我们要传入的参数是 CTA rank 和 CTA cluster 布局，而不是线程序号。

```python
        a_cta_layout = cute.make_layout(cute.slice_(cta_layout_mnk, (0, None, 0)).shape)
        a_cta_crd = cluster_coord_mnk[1]
        sA_for_tma_partition = cute.group_modes(sA, 0, 2)
        gA_for_tma_partition = cute.group_modes(gA_mkl, 0, 2)
        tAsA, tAgA_mkl = cute.nvgpu.cpasync.tma_partition(
            tma_atom_a,
            a_cta_crd,
            a_cta_layout,
            sA_for_tma_partition,
            gA_for_tma_partition,
        )
```

## 初始化 mbarrier

第 5 步初始化 mbarrier。虽然我们可以借助`PipelineTmaAsync`，但仍然需要计算大量的参数。在这个场景中，这些参数是完全固定的，只不过 CuTe DSL 没有提供相关的函数。这一步我们需要额外考虑的参数包括：
- TMA 传输的总字节数，包括这个 mbarrier 处理的所有的 TMA 传输的字节数，也就是 A 和 B 的传输数量都需要算进去。
- mbarrier 到达次数，这个数量和`PipelineTmaAsync`内部实现有关，具体计算后续介绍。
- CTA cluster 布局。CTA rank 在`PipelineTmaAsync`内部会自己读取，不用指定。
- `PipelineTmaAsync`为了方便，一个对象就处理了多级流水的多个 mbarrier，所以还需要指定流水级数。

```python
        # 计算传输大小
        a_smem_layout = cute.slice_(a_smem_layout_staged, (None, None, 0))
        b_smem_layout = cute.slice_(b_smem_layout_staged, (None, None, 0))
        tma_copy_bytes = cute.size_in_bytes(self.a_dtype, a_smem_layout) + cute.size_in_bytes(self.b_dtype, b_smem_layout)

        # TMA 生产者和消费者到达次数
        mainloop_pipeline_producer_group = pipeline.CooperativeGroup(
            pipeline.Agent.Thread, 1
        )
        # self.num_mcast_ctas_a: self.cluster_shape_mn[1]
        mcast_size = self.num_mcast_ctas_a + self.num_mcast_ctas_b - 1
        num_warps = self.threads_per_cta // 32 # self.threads_per_cta: 128 or 256
        consumer_arrive_cnt = mcast_size * num_warps
        mainloop_pipeline_consumer_group = pipeline.CooperativeGroup(
            pipeline.Agent.Thread, consumer_arrive_cnt
        )

        # CTA cluster 布局
        cta_layout_vmnk = cute.make_layout((1, *cta_layout_mnk.shape))
        mainloop_pipeline_array_ptr = storage.mainloop_pipeline_array_ptr.data_ptr()
        mainloop_pipeline = pipeline.PipelineTmaAsync.create(
            barrier_storage=mainloop_pipeline_array_ptr,
            num_stages=self.ab_stage,
            producer_group=mainloop_pipeline_producer_group,
            consumer_group=mainloop_pipeline_consumer_group,
            tx_count=tma_copy_bytes,
            cta_layout_vmnk=cta_layout_vmnk,
        )

        # 初始化 mbarrier 后进行同步
        if cute.size(self.cluster_shape_mn) > 1:
            cute.arch.cluster_arrive_relaxed()
            cute.arch.cluster_wait()
        else:
            cute.arch.sync_threads()
```

## 发射 TMA 指令

这一部分在主循环内执行，在`PipelineAsync`必备的 6 步中的第 2 步调用`cute.copy`发射 TMA 指令。
与其他的`cute.copy`相比，需要提供 mbarrier 的指针，并且在多播时还要计算一个 mask。多播 mask 是一个 16 位整数（CTA cluster 大小不会超过 16），表示这次 TMA 读取要写到 CTA cluster 中哪些 CTA，例如一整行或者一整列 CTA。多播 mask 直接调用`cute.make_layout_image_mask`即可。

需要注意，虽然 mbarrier 和 TMA 都是单线程语义，但是**`cute.copy`和`PipelineAsync`的都是 warp 级别，内部实现会选出一个（或多个）线程完成**。

```python
        # 0. 计算多播 mask
        a_mcast_mask = cute.make_layout_image_mask(
            cta_layout_mnk, cluster_coord_mnk, mode=1
        ) if self.is_a_mcast else 0
        # 主循环
        mainloop_producer_state = pipeline.make_pipeline_state(pipeline.PipelineUserType.Producer, self.ab_stage)
        mainloop_consumer_state = pipeline.make_pipeline_state(pipeline.PipelineUserType.Consumer, self.ab_stage)
        for k_tile in cutlass.range(k_pipe_mmas, k_tile_cnt, 1, unroll=1):
            # 4. 消费者：等待 TMA 传输完成
            mainloop_pipeline.consumer_wait(mainloop_consumer_state)
            # 5. 消费者：计算...
            # 6. 消费者：计算后，释放缓冲区
            mainloop_pipeline.consumer_release(mainloop_consumer_state)
            mainloop_consumer_state.advance()

            # /////////////////////////////////////////////////////////////////////////////
            #  TMA load
            # /////////////////////////////////////////////////////////////////////////////
            # 注意是 warp 级别！
            if warp_idx == 0 and mainloop_producer_state.count < k_tile_cnt:
                # 1. 生产者：等待消费者释放缓冲区
                mainloop_pipeline.producer_acquire(mainloop_producer_state)
                # count 不循环，index 循环
                tAgA_k = tAgA_mkl[(None, mainloop_producer_state.count)]
                tAsA_pipe = tAsA[(None, mainloop_producer_state.index)]

                # 2. 生产者：发射 TMA 指令
                mbar_ptr = mainloop_pipeline.producer_get_barrier(mainloop_producer_state)
                cute.copy(
                    tma_atom_a,
                    tAgA_k,
                    tAsA_pipe,
                    tma_bar_ptr=mbar_ptr,
                    mcast_mask=a_mcast_mask,
                )
                # 3. TMA 异步完成传输，所以 producer_commit 本身是 noop
                mainloop_pipeline.producer_commit(mainloop_producer_state)
                mainloop_producer_state.advance()
```

## `PipelineTmaAsync`内部实现

前面第 5 步中，生产者和消费者的到达次数**并不是可调的参数，而是具体同步实现的抽象泄露**。所以必须考查`PipelineTmaAsync`的内部实现：

- 消费者的 release 和 wait，一个是到达一个是等待。到达操作是有条件的并且每个线程的`consumer_mask`表示到达的 peer CTA rank。
- 生产者的 acquire 负责等待并且重置 TMA 传输计数器，commit 实际上是 TMA 完成的，所以对应的 CUDA 核是 noop。

生产者部分的同步大部分完整都讲过了，而且 TMA GMEM->SMEM 传输完成是基于传输字节数，与是否多播无关（可能也是因此只支持这个同步方法）。在多播的情况下，消费者部分的同步需要特别考虑。

CTA cluster 范围的同步方法有两种：

- 基于`cluster_arrive`和`cluster_wait`，这个方法类似`__syncthreads`，只能做全 CTA cluster 范围内的同步，无法做细粒度控制。（好像未尝不可？求解答）
- mbarrier 的到达操作可以选择 cluster 中别的 CTA 作为目标，因此可以在每个 CTA 执行到达时，以所有相关的 CTA 中对应的 mbarrier作为目标执行到达，用于通知该缓冲区可以施放。例如多播 A 时，一个消费者到达了，通知同一行所有的生产者；而多播 B 时，通知同一列所有的生产者。

为了完成这个任务，`create`函数中读取 CTA rank 和线程号，并用`init_empty_barrier_arrive_signal`函数计算消费者中哪个线程负责通知哪个 CTA。根据计算结果，每个 warp 中，对于每个目标 CTA，都选出一个线程用于执行到达操作。

一个生产者要想要开始发射 TMA 操作，必须要等待他要多播 A 矩阵的所有的 CTA 都释放缓冲区，并且等待所有他要多播 B 的 CTA 也都释放缓冲区。这样算出来，每个生产者会收到`self.num_mcast_ctas_a + self.num_mcast_ctas_b - 1`个 CTA 的到达操作，这么多个 CTA 的每个 warp 各执行一次。至于为什么要减 1，是因为本身的 CTA 算了两次。可以参考下图。

![Multicast layout](https://i0.wp.com/research.colfax-intl.com/wp-content/uploads/2025/05/image-4-scaled.png?resize=1536%2C501&ssl=1)

```python
@dataclass(frozen=True)
class PipelineTmaAsync(PipelineAsync):
    def producer_acquire(
        self, state: PipelineState, try_acquire_token: Optional[Boolean] = None
    ):
        if_generate(
            try_acquire_token is None or try_acquire_token == 0,
            lambda: self.sync_object_empty.wait(state.index, state.phase),
        )
        self.sync_object_full.arrive(state.index, self.producer_mask)

    def producer_commit(self, state: PipelineState):
        pass

    def consumer_release(self, state: PipelineState):
        if_generate(
            self.is_signalling_thread,
            lambda: self.sync_object_empty.arrive(state.index, self.consumer_mask),
        )

    # 继承自PipelineAsync
    def consumer_wait(
        self, state: PipelineState, try_wait_token: Optional[Boolean] = None
    ):
        if_generate(
            try_wait_token is None or try_wait_token == 0,
            lambda: self.sync_object_full.wait(state.index, state.phase),
        )
```

`PipelineTmaAsync`只是一组`SyncObject`的封装，实际上到达和等待操作是使用`MbarrierArray`进行。
这个类也不是一个好惹的角色，其中`arrive`操作对于 TMA 有一个特化，**会先用`elect_one()`从一个warp中挑一个线程，再调用`mbarrier_arrive_and_expect_tx`**，到达的同时更新 TMA 传输字节数（expect_tx）。这可以解释为什么生产者的线程数是 1，并且对应的操作是 warp 级。

> mbarrier 作为一个 64 位整数，里面[只能放 3 个 2^20 大小的数](https://docs.nvidia.com/cuda/parallel-thread-execution/#parallel-synchronization-and-communication-instructions-mbarrier-contents)。其中总到达数和当前到达数不可或缺，已经占了两个，只剩一个位置，没法同时把总传输数和当前传输数都存进来了。因此剩下的一个位置放的 tx-count 实际上是指**总传输数-当前传输数**，需要每次循环都由一个线程在`producer_acquire`设置一次。

```python
class MbarrierArray(SyncObject):
    def arrive(
        self,
        index: int,
        dst: int,
        cta_group: Optional[cute.nvgpu.tcgen05.CtaGroup] = None,
    ) -> None:
        if self.op_type is PipelineOp.AsyncThread:
            self.arrive_mbarrier(index, dst)
        elif self.op_type is PipelineOp.TCGen05Mma:
            assert cta_group is not None, (
                "Error: CTA group must be provided for TCGen05Mma."
            )
            self.arrive_tcgen05mma(index, dst, cta_group)
        elif self.op_type in [PipelineOp.TmaLoad]:
            self.arrive_and_expect_tx(index, self.tx_count)
        elif self.op_type is PipelineOp.AsyncLoad:
            self.arrive_cp_async_mbarrier(index)
        else:
            assert False, (
                f"Error: MbarrierArray is not supported for PipelineOp: {_get_pipeline_op(self.op_type)}."
            )

    def arrive_mbarrier(self, index: int, dst_rank: Optional[int] = None) -> None:
        if dst_rank is None:
            cute.arch.mbarrier_arrive(self.get_barrier(index))
        else:
            cute.arch.mbarrier_arrive(self.get_barrier(index), dst_rank)

    def arrive_and_expect_tx(self, index: int, tx_count: int) -> None:
        with cute.arch.elect_one():
            cute.arch.mbarrier_arrive_and_expect_tx(self.get_barrier(index), tx_count)

    def wait(self, index: int, phase: int) -> None:
        cute.arch.mbarrier_wait(self.get_barrier(index), phase)
```

总之，我们发现：

- 生产者的到达数是 1。
- 消费者的到达数是多播涉及的 CTA 数，再乘上每个 CTA 到达的数量。

# 总结

相比于非常简洁的基本 TMA，涉及到多播时，整个操作需要 TMA 和同步算法的紧密配合，因此上面的分析也仅对`PipelineTmaAsync`成立，这可能也是 CuTe DSL 没有给出比较简练的封装的原因。另外，本文只是代码阅读笔记，没有来得及校验上述分析的准确性，如有遗漏，希望大佬们指正。
