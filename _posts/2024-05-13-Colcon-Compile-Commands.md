---
layout: post
title: ROS2 colcon编译器下生成compile_commands.json
date: 2024-05-13 22:00:00 +0800
tags: 编程
render_with_liquid: false
---

`compile_commands.json` 是 VS Code 里面阅读 C++ 的利器，通过指定 `compile_commands.json` 可以配置 C++ 代码分析需要的大部分参数，[启动 Intellisense](https://code.visualstudio.com/docs/cpp/configure-intellisense#_project-level-intellisense-configuration)。

CMake 中，可以使用 [CMAKE_EXPORT_COMPILE_COMMANDS](https://cmake.org/cmake/help/latest/variable/CMAKE_EXPORT_COMPILE_COMMANDS.html) 这个 CMake 变量或对应的环境变量来生成 `compile_commands.json`，即执行 `cmake` 命令时添加选项 `-DCMAKE_EXPORT_COMPILE_COMMANDS=ON`，或是添加环境变量 `export CMAKE_EXPORT_COMPILE_COMMANDS=ON`。

在 ROS1 中，通过修改`CMakeCache.txt`即可导出`compile_commands.json`。
但 ROS2 的 colcon 会默认设置 CMake 变量 `CMAKE_EXPORT_COMPILE_COMMANDS` 为不导出 `compile_commands.json`，无法使用环境变量的方法配置，必须添加参数，如

```shell
colcon build --cmake-args -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
```

注：`colcon build --cmake-args` 和 `/usr/bin/env -S` 不同，不执行分词，而是把后续所有不认识的选项都作为 CMake 参数，如添加多个参数时需要使用：

```shell
colcon build --cmake-args -DCMAKE_EXPORT_COMPILE_COMMANDS=ON -DPYTHON_EXECUTABLE=/usr/bin/python3
```
