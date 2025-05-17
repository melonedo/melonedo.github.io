---
layout: post
title: CMake 调试指南
date: 2024-11-02 22:24:00 +0800
tags: 
- 编程
- CMake
render_with_liquid: false
---

C/C++ 由于诞生在还没有大规模项目管理的洪荒年代，各个项目间的依赖关系向来依赖人力排查。而在最近的十年，CMake 逐渐建立了生态，成为了现在主流 C 项目的事实标准。然而，CMake 诞生之初并没有机会仔细考虑依赖处理的问题，同时加上当年的脚本语法尚未收敛，导致 CMake 项目一旦出错，则难以调试。本文对于常见的问题及调试方法做一个简单的说明。

## CMake 简介

由于 CMake 的普及，大部分的 C 项目都会在顶层放上一个 CMakeLists.txt，可以用以下方法编译：

```shell
cmake -B build
cmake --build build
```

当然，上述两个命令需要大约 18 年后的 CMake 才支持，为了兼容性，很多项目会写成

```shell
mkdir build
cmake ..
make
```

在 2024 年的今天，后者带来的兼容性优势已经可以忽略。无论如何，我们可以看到，一个使用 CMake 的项目编译都分为两步：

1. 配置项目，生成工程文件（`cmake`），在 Linux 上默认生成 Makefile，在 Windows 默认生成 VS 解决方案（.sln）。
2. 根据工程文件进行编译（`cmake --build`或是`make`）

也就是说，CMake 在 Linux 上是生成一个 Makefile，再进一步调用`make`，再由`make`组织具体的编译过程。

而 CMake 本身也被很多工具套娃，比如 ROS 所用的 catkin/colcon 都依赖于 CMake 配置项目。

要注意的是，为了方便调试，之前配置的部分结果会保存在`build/CMakeCache.txt`中，而且 CMakeLists.txt 本身也作为工程文件中的一个依赖，执行`cmake --build`时，如果发现 CMakeLists.txt 更新，也会自动运行一次`cmake`。由于自动运行`cmake`时的参数和环境等不易控制，因此本文建议，在不熟悉的情况下，**每当修改 CMakeLists.txt 后，删除`build/CMakeCache.txt`，避免自动运行 CMake**。

## 找不到包

> TODO: find_package  `--debug-find`

## 找错库

> TODO: target_link_libraries 看 CMakeCache.txt 里面库的名字，`--verbose`和ldd确认

