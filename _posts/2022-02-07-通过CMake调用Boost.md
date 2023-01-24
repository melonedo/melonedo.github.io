---
layout: post
title: 通过CMake调用Boost
date: 2022-02-07 19:01:42 +0800
tags:
- 编程
- CMake
---

Boost是C++中非常常用的库，Boost的[官方文档](https://www.boost.org/doc/libs/1_78_0/more/getting_started/index.html)中没有提到如何使用cmake调用，这里简单说明操作步骤。

## 从源代码安装Boost

包管理器很多都配备了Boost，但Windows上这并不容易，于是我们介绍从源代码安装的方法。

1. 首先从官网下载[源码](https://www.boost.org/users/download/)并解压到路径`path/to/boost_xxx`（后面会用到）。
2. 在`path/to/boost_xxx`目录中运行`./bootstrap`然后运行`./b2 --prefix=install_dir install`。

至此已经安装完毕，其实做第二步主要是生成CMake模组文件。由于CMake对Boost的支持是内置的，最好CMake比对应版本的Boost晚发布，这样使用体验是最好的。如果CMake低于3.5，可以参考[这个文章的方法](https://cliutils.gitlab.io/modern-cmake/chapters/packages/Boost.html)。

注：Windows下提供[预编译的Boost](https://sourceforge.net/projects/boost/files/boost-binaries/)。这个库的静态库命名似乎有点奇怪，需要手动关掉自动连接。

## 通过CMake调用Boost

以一个利用Boost::thread的简单程序为例，C++代码

```c++
// test.cpp
#include <boost/thread/thread.hpp>

int main() {
  boost::thread t;
  t.join();  // do nothing
  return 0;
}
```

参考[FindBoost](https://cmake.org/cmake/help/latest/module/FindBoost.html#boost-cmake)的文档，对应的CMake模板

```cmake
cmake_minimum_required(VERSION 3.9)
project(boost_test)
find_package(Boost REQUIRED COMPONENTS thread)

add_executable(test test.cpp)
# 用CMake的时候不需要用#comment(lib, "...")自动连接，而且这样有时会自动给静态库加lib前缀，不适配某些版本（如1.74）的Windows下预编译库的名字。
target_link_libraries(test Boost::thread Boost::disable_autolinking Boost::diagnostic_definitions)
```

CMake生成的时候需要通过`BOOST_ROOT`变量找到刚才编译Boost的路径`path/to/boost_xxx/install_dir`（install_dir是前面`--prefix=xxx`配置的安装路径），如

```shell
# -B需要cmake 3.13以上
cmake -D BOOST_ROOT=path/to/boost_xxx/install_dir -B build
cmake --build build
```

### 目标名

纯头文件的目标统一是`Boost::boost`，其他的目标通常就是库的名字。`path/to/boost/install_dir/lib/cmake`里面没有的目标就是纯头文件库。

### 关于提示Please define `_WIN32_WINNT` or `_WIN32_WINDOWS`

可以在文件最开头加上
```c++
#include <SDKDDKVer.h>
```

或者用CMake预编译头的方法强制在每个源的最开头都加上这个文件：

```cmake
if(MSVC)
  target_precompile_headers(test PRIVATE <SDKDDKVer.h>)
endif()
```
