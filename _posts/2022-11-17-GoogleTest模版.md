---
layout: post
title: GoogleTest 模版 CMakeLists
date: 2021-11-17 08:09:16 +0800
tags:
- 编程
- CMake
---

- 项目是动态库MyMatrix
- 在windows下自动导出所有符号，使用utf-8编码
- ~~使用GitHub反代fastgit.org~~（已失效）
- 自动下载GoogleTest，不需要额外安装
- 自动设置vs启动项

```cmake
# https://google.github.io/googletest/quickstart-cmake.html#create-and-run-a-binary
cmake_minimum_required(VERSION 3.14)
project(MyMatrix)

if(MSVC)
    add_definitions(-D_CRT_SECURE_NO_WARNINGS)
    # 更优雅的方法参见CUDA那篇
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /utf-8")
endif()

set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS true)
add_library(MyMatrix SHARED MyMatrix.cpp)

option(BUILD_GMOCK OFF)
include(FetchContent)
FetchContent_Declare(
  googletest
  # Specify the commit you depend on and update it regularly.
  URL https://github.com/google/googletest/archive/refs/tags/release-1.11.0.zip
)
# For Windows: Prevent overriding the parent project's compiler/linker settings
set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(googletest)

add_executable(tests MyMatrix-tests.cpp)
target_link_libraries(tests gtest_main)
enable_testing()
add_test(NAME tests COMMAND tests)
set_property(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} PROPERTY VS_STARTUP_PROJECT tests)
```

常见cmake指令：

- 生成：`cmake -B build`
- 编译：`cmake --build build`
- 打开（vs解决方案）：`cmake --open build`

MyMatrix-tests.cpp内容参见[Quickstart: Building with CMake | GoogleTest](https://google.github.io/googletest/quickstart-cmake.html#create-and-run-a-binary)。
