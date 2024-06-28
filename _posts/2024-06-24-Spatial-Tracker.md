---
layout: post
title: Spatial Tracker 复现笔记
date: 2024-06-24 00:09:00 +0800
tags: 
- 编程
render_with_liquid: false
---

[SpaTracker](https://github.com/henry123-boy/SpaTracker) 是一个很有趣的像素极目标追踪库。这里记录一下复现步骤。

## 下载

本文使用的提交是 4e777d1。克隆后，还需要下载：

- SpaTracker 权重及示例：<https://drive.google.com/drive/folders/1UtzUJLPhJdUg2XvemXXz1oe6KUQKVjsZ?usp=sharing>
- MiDaS 权重：<https://github.com/isl-org/MiDaS/releases/tag/v3_1>
- ZoeDepth 权重：<https://github.com/isl-org/ZoeDepth/releases/tag/v1.0>

## 安装

首先新建环境

```
conda create -n SpaTrack python==3.10
conda activate SpaTrack
```

后面的 cupy 本地编译可能出错，改为 conda 安装。安装前，注释掉 requirements.txt 中 cupy 一行。

```
conda install -c conda-forge cupy=12.2 cuda-version=11.8
```

这样安装完成后，如果环境里的 libstdc++ 比较旧，可能会出现问题。改用 conda 提供的可以解决。<https://stackoverflow.com/questions/72540359/glibcxx-3-4-30-not-found-for-librosa-in-conda-virtual-environment-after-tryin>

```
conda install -c conda-forge libstdcxx-ng=12
```

最后再装 pip 包。注意需要严格按照先 conda 再 pip 的顺序，如果弄错了就 `conda remove -n SpaTracker --all`删掉环境重新开始。否则报错“It appears that Pytorch has loaded the ‘torch/_C’ folder”。<https://github.com/pytorch/pytorch/issues/62470>。（这个`--all`是删除环境内所有包的意思）

```
pip install torch==2.1.1 torchvision==0.16.1 torchaudio==2.1.1 --index-url https://download.pytorch.org/whl/cu118
# 注意 requirements.txt 里面的 cupy 要删掉
pip install -r requirements.txt
```

## 运行

```
conda activate SpaTracker
# for fish
# set -xp LD_LIBRARY_PATH (dirname $CONDA_EXE)/../envs/$CONDA_DEFAULT_ENV/lib
# for bash
export LD_LIBRARY_PATH="$(dirname $CONDA_EXE)/../envs/$CONDA_DEFAULT_ENV/lib:LD_LIBRARY_PATH"
# --vid_name butterfly --root examples/butterfly_rgb 表示视频为 examples/butterfly_rgb/butterfly.mp4
pyton demo.py --model spatracker --downsample 1 --vid_name butterfly --len_track 1 --fps_vis 15  --fps 1 --grid_size 40 --root examples/butterfly_rgb
```

