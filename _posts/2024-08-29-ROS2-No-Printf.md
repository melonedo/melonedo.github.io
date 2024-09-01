---
layout: post
title: ROS 2 没有 printf 输出
date: 2024-08-29 00:15:38 +0800
tags: 
- 编程
render_with_liquid: false
---

ROS2 中调试经常发现 printf 并不会输出到屏幕上，[No stdout logging output in ROS2 using launch
](https://answers.ros.org/question/332829/no-stdout-logging-output-in-ros2-using-launch/)。这是因为 ros2 launch 把输出缓冲了，解决方法是添加一个`emulate_tty=True`
```python
Node(
    package='package_name',
    node_executable='package_exec',
    output='screen',
    emulate_tty=True,
    arguments=[('__log_level:=debug')]
)
```
