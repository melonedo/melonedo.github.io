---
layout: post
title: ROS1中的参数传递
date: 2024-11-02 15:28:00 +0800
tags: 
- 编程
- ROS
render_with_liquid: false
---

一个程序运行时，通常需要获取配置一些和运行环境相关的参数，如一个摄像头。那么除了在编译时就让程序自动获取对应参数，或是硬编码参数以外，就需要在运行时获取对应的参数。通常，公认的参数获取方法包括：

- 通过图形或者是命令行界面和用户直接发起交互
- 使用命令行参数，比如`ls -l`中的`-l`或是`cmake --build build`中的`--build build`
- 从环境变量获取参数，比如使用`wget`时会参考`https_proxy`变量
- 从配置文件获取参数，比如 .bashrc/.profile 等

选择使用哪种参数获取方式取决于软件的使用场景，比如 systemd 不由用户主动启动，而启动的环境也由系统指定，因此只能从配置文件中获取参数。而需要用户输入密码的程序为了安全，一般不会存储密码，需要通过用户交互获取参数。而没有特殊要求的程序通常会结合多种来源的参数，比如 CMake 除了读取编写的 CMakeLists.txt 以外，还会从命令行中读取各种参数（一般是`-DXX=YY`），并且还会综合环境变量中包括`PATH/CC/OpenCV_DIR`等各种各样变量的内容进行综合判断。

## rosrun

而在 ROS 中，由于大部分程序并不由用户直接启动，而是以 .launch 文件的形式组织，因此参数获取同样是关键的问题。作为生态的一部分，ROS1 中直接规定了以 [rosrun](http://wiki.ros.org/rosbash#rosrun) 形式运行的 ROS 节点的参数应是[以下形式](https://wiki.ros.org/Remapping%20Arguments)：

- `from:=to`：[话题映射](https://wiki.ros.org/Remapping%20Arguments#Remapping_Arguments)，将节点代码中的`from`话题定义到外界的`to`话题。一个常见的情况 [image_transport](https://wiki.ros.org/image_transport) 只会订阅`in`话题并发布`out`话题，使用`rosrun image_transport republish raw in:=camera/image out:=camera/image_repub`的形式在运行时定义为外界的话题。
- `_param:=value`：定义节点的[私有参数](https://wiki.ros.org/Parameter%20Server#Private_Parameters)，也就是只对该节点有效，不是全局生效的参数。通过这个方法定义的参数和后面 roslaunch 中的`<param>`标签是同样的，在程序中新建`ros::NodeHandle("~")`，再使用`ros::NodeHandle::param`方法读取。注意`~`表示当前节点，如果读取全局参数使用`ros::NodeHandle`的默认构造函数即可

当然，作为一个完整的程序，ROS 节点同样可以使用包括环境变量、配置文件在内的其他的方法获取参数，并不需要来自 rosrun 的支持。比如环境变量[`ROSCONSOLE_FORMAT`](https://wiki.ros.org/ROS/EnvironmentVariables#Console_Output_Formatting)可以用来很方便地修改节点的日志内容。

## roslaunch

既然名叫节点，那么 ROS 的常见用法理应是同时运行多个节点，用于组成一个完整的图。ROS 为此任务提供了专门的工具 [roslaunch](http://wiki.ros.org/roslaunch)。roslaunch 并不是一个用户提供的程序，因此 roslaunch 中只能使用固定的几种方法获取参数：

- 首先自然是 [.launch](http://wiki.ros.org/roslaunch/XML) 文件，以 XML 格式编写，其中根元素是`<launch>`标签，其中包含若干个[`<node>`](http://wiki.ros.org/roslaunch/XML/node)用于运行`rosrun`，或是[`<include>`](http://wiki.ros.org/roslaunch/XML/include)重用其他的 .launch 文件。使用[`<arg>`](http://wiki.ros.org/roslaunch/XML/arg)标签声明变量（为了区分，本文把`<arg>`声明的叫做变量）。**注意，roslaunch 并不使用 [rosparam](http://wiki.ros.org/rosparam) 机制，`<arg>`参数只能在本文件中以`$(arg var)`的形式使用**。
- 从其他来源获取的参数，通过[参数替换](http://wiki.ros.org/roslaunch/XML#substitution_args)的方法使用。如果需要读取环境变量，可以使用`$(env ENV)/$(optenv ENV)`，而读取变量使用`$(arg var)`。另外，`$(find pkg)`用于查找包的路径，也非常常用。
- 在命令行，可以使用`roslaunch package file.launch arg:=value`的形式赋值上述已经通过`<arg>`声明的变量。**注意，roslaunch 变量赋值的形式和`rosrun`类似，但含义截然不同**。例如，roslaunch 命令无法建立话题映射。

在一个`<include>`或是`<arg>`元素内部，还可以嵌套[`<param>`](http://wiki.ros.org/roslaunch/XML/param)、[`<remap>`](http://wiki.ros.org/roslaunch/XML/remap)、[`<arg>`](http://wiki.ros.org/roslaunch/XML/arg) 等元素，用于传递参数、话题映射、变量等。
