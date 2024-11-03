---
layout: post
title: CMake中手动指定Python路径
date: 2024-05-11 21:01:00 +0800
tags: 
- 编程
- CMake
render_with_liquid: false
---

众所周知，Python经常会在一台电脑里装很多个版本。有虚拟环境时每个虚拟环境都有一个对应的解释器，而即使没有虚拟环境，conda等软件也会安装自己的Python。因此，在使用CMake编译时，默认的Python可能不是我们想要的。


根据CMake依赖的搜索方式的不同，手动指定Python版本时，需要设定CMake变量：

 - `Python_EXECUTABLE`([FindPython](https://cmake.org/cmake/help/latest/module/FindPython.html))
 - `Python3_EXECUTABLE`([FindPython3](https://cmake.org/cmake/help/latest/module/FindPython3.html))
 - `PYTHON_EXECUTABLE`([FindPythonInterp](https://cmake.org/cmake/help/latest/module/FindPythonInterp.html))

要注意的是这些变量都是CMake变量，不是环境变量，只能在执行CMake时添加选项（或者更改CMakeCache.txt）来指定，如`-DPython3_EXECUTABLE=/usr/bin/python3`。