---
layout: post
title: Segment Anything 平替
date: 2025-05-17 19:00:00 +0800
tags:
- AI
render_with_liquid: false
---

视觉领域能够落地能用的模型不多，[Segment Anything](https://github.com/facebookresearch/segment-anything)（SAM） 是少数效果非常惊艳的模型。然而原版 SAM 基于特别大的视觉 Transformer（ViT），非常难以部署。

因此后续有大量的改进，例如 [On Efficient Variants of Segment Anything Model: A Survey](https://arxiv.org/abs/2410.04960) 就总结了大部分上述的模型，不过没有什么见解。知乎文章 [Segment Anything(SAM)的哪些后续方法，又快又好？](https://zhuanlan.zhihu.com/p/683729749) 简要地介绍了不少其中的模型。这里我总结几个看起来比较帕累托最优的，以后有空试试。

## [MobileSAM](https://arxiv.org/abs/2306.14289) 和 [EfficientSAM](https://arxiv.org/abs/2312.00863)

这两个都是直接缩小 ViT 的尺寸做出来的，前者发得早，知名度高，后者是 Meta 自己出的，方法和 MobileSAM 差不多。注意 EfficientSAM 引用了 [TinyViT](https://arxiv.org/abs/2207.10666)，但实际上用的是 [DeiT](https://arxiv.org/abs/2012.12877) 的模型参数，前者有一堆省计算量的小技巧。

## [EfficientViT-SAM](https://arxiv.org/abs/2402.05008)

直接把注意力换成 ReLU 注意力，全连接换成卷积，是一种线性注意力。不过实际上 Token 数量不多，应该也省不了多少计算量，不过既然是一种注意力，根据 [MetaFormer 理论](https://arxiv.org/abs/2111.11418)，只要能做 Token 交互的算子都能用，问题应该不大。

EfficientVIT 有重名，这里用的是 MIT Han Lab 的 [ReLU 注意力版本](https://arxiv.org/abs/2205.14756)，不是[级联注意力的版本](https://arxiv.org/abs/2305.07027)。级联注意力把多个头并行改为了前一个头的输出加到下一个头的输入上，节省了内存，相当于展成了超深的神经网络，对计算应该非常不友好。

## [RepViT-SAM](https://arxiv.org/abs/2312.05760) 和 [EdgeSAM](https://arxiv.org/abs/2312.06660)

两个都是基于 [RepViT](https://arxiv.org/abs/2307.09283) 这个 CNN 模型的，看起来效果还可以，应该会比这些 Transformer 好部署不少。RepViT-SAM 是 RepViT 的作者自己发的，不过不如 EdgeSAM 做得多。
