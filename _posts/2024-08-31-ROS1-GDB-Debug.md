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

如果直接在命令行调试比较难受，则可以用 GDB Server 远程（参考 [How to attach to remote gdb with vscode?](https://stackoverflow.com/questions/53519668/how-to-attach-to-remote-gdb-with-vscode)）

首先把`launch-prefix`改为`gdbserver localhost:10000`。

然后在 .vscode/launch.json 添加调试配置
```json
{
    // Use IntelliSense to learn about possible attributes.
    // Hover to view descriptions of existing attributes.
    // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Debug listener",
            "type": "cppdbg",
            "request": "launch",
            "program": "/path/to/listener", // for loading symbols from running program
            "cwd": "${workspaceFolder}",
            // if you want to connect at entry point (requires remote program to start paused)
            "stopAtEntry": false,
            "stopAtConnect": false,
            "MIMode": "gdb",
            "miDebuggerPath": "/usr/bin/gdb",
            "miDebuggerServerAddress": "localhost:10000",
            "setupCommands": [
                {
                    "description": "Enable pretty-printing for gdb",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true,
                }
            ]
        }
    ]
}
```