---
layout: post
title: ROS 1 GDB 调试方法
date: 2024-08-31 00:13:18 +0800
tags: 
- 编程
render_with_liquid: false
---

ROS 1 的代码中，如果 c++ 部分崩溃，直接 gdb 并不容易：
[How to Roslaunch Nodes in Valgrind or GDB](http://wiki.ros.org/roslaunch/Tutorials/Roslaunch%20Nodes%20in%20Valgrind%20or%20GDB)。

比较好的办法是在 .launch 中对应的`<node />`添加`launch-prefix="gdb -ex run --args"`，如

```xml
<node name="listener" pkg="examples" type="listener" output="screen" launch-prefix="gdb -ex run --args" /> 
```
