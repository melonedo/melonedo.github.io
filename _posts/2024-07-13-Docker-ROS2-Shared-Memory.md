---
layout: post
title: ROS 2 在 Docker 启用共享内存传输
date: 2024-07-13 21:44:00 +0800
tags: 
- 编程
render_with_liquid: false
---

ROS 2 相比 ROS 1 的一个重大改进是直接复用了现成的 DDS 中间件，而不再自己定义中间件。默认的情况下，ROS 2 会使用 [Fast DDS](https://fast-dds.docs.eprosima.com/) 作为中间件，提供 DDS 协议，即话题发布和接收的功能。

DDS 协议据 [ROS 2 设计博客](https://design.ros2.org/articles/ros_on_dds.html)说已经身经百战，质量稳定，但实际上由于 DDS 协议是去中心化的协议，不容易直接地诊断网络问题，并且 DDS 协议栈也比较复杂，调试起来很困难。

![Fast DDS 协议栈](https://fast-dds.docs.eprosima.com/en/stable/_images/library_overview.svg)

对于 ROS 1 而言，进行基本的通信只需要指定一个`ROS_MATER_URI`即可，所有的数据通信都由中心化的节点管理。不过，在运行`rosnode info`时，还是会需要直接访问单个节点自身的端口，但无伤大雅。

# ROS 2 发现节点

而 ROS 2 没有中心的节点，默认使用的是直接 UDP 多播的方式发现一个网络内的节点，而不同的网络通过[`ROS_DOMAIN_ID`](https://docs.ros.org/en/jazzy/Concepts/Intermediate/About-Domain-ID.html)进行区分。另外，也可以用一个中心化的[发现服务器](https://docs.ros.org/en/jazzy/Tutorials/Advanced/Discovery-Server/Discovery-Server.html)组织节点，但实际的数据传输不会走这个发现服务器，意义不大。

## UDP 多播

如果一开始的`ros2 node list`无法找到所有节点，ROS 2 提供了一个默认的工具`ros multicast`检查能否正常发现节点：

```bash
# 在一个网络中
ros2 multicast receive
# 在另一个网络中
ros2 multicast send
```
如果两个网络支持 UDP 多播，则这两个设备中的节点可以互相发现。在 Docker 中，默认会为容器建立独立的网络，这些网络除了 IP 不同以及具有防火墙相关设置外，和直接回环区别不大。如果希望直接用回环网络，也可以指定`--network host`。但使用回环网络时，会遇到后面的问题。

# ROS 2 数据传输

但是，ROS 2 的一个非常麻烦的地方在于，数据传输和发现节点是独立的。ROS 2 默认的中间件 Fast DDS 默认的传输层配置 [DEFAULT](https://fast-dds.docs.eprosima.com/en/stable/fastdds/env_vars/env_vars.html#fastdds-builtin-transports) 是：

1. 使用 IPv4 UDP 发现节点。
2. 随后，**如果两个节点的 IP 相同**，则使用共享内存传输，否则使用 IPv4 UDP 传输。

Fast DDS 的判断非常地简单，不会具体地在共享内存里进行一次传输作为实验。这导致的结果是，如果两个节点间存在可以发现但不可以传输的状态，即`ros2 topic list`或者`ros2 node list`可以看到节点，**但`ros2 topic echo`没有输出**。

## 共享内存

共享内存的性能显然要高于回环 TCP，而 UDP 由于需要大量地拆包解包，性能更比 TCP 差，我的设备上可能只能跑到 TCP 1/4 以下的性能，总的带宽只略高于 1Gb/s，对于大数据量的任务显然无法胜任。因此，默认使用共享内存是非常合理的。

共享内存是 Linux 中一个非常独特的文件目录，位于`/dev/shm`下，也可以通过`ipcs`指令查看，可以参考[Shared Memory & Docker](https://datawookie.dev/blog/2021/11/shared-memory-docker/)。而作为文件，共享内存文件也同样要受到文件权限的限制。通过`ls -l /dev/shm`可以看到。

## 文件权限

Linux 中，文件的权限分为几部分：

- 创建者自己的权限
- 对应用户组的权限
- 其他人的权限

其中，创建者和用户组的区分是根据一个整数 UID/GID 进行的。一般直接读取整数比较困难，因此 /etc/passwd 中需要存放每个用户的用户名和对应 UID，`ls -l`在显示时会查找对应用户名并显示。

## Docker 用户权限

Docker 中，用户权限的管理非常复杂，默认情况下直接使用 root 用户，这也对应 dockerd 需要用 root 运行。但也可以用`-u`指定特定用户，或是指定特定命名空间。
这里需要注意的是，在不使用命名空间时，Docker 内外的 UID 是通用的，即类似于端口号。因此，指定的用户实际上就是在 Docker 外的实际用户。

# Docker 配置

那么，在 Docker 内外实现共享内存通信的要素都拼凑齐了。

- 使用`--network host`直接使用宿主网络，让 Fast DDS 知道这是同一台电脑。一般使用 Docker 时都使用宿主网络，毕竟隔离也没什么意义。
- 使用`--ipc host`共享宿主和容器内的 IPC 资源，包括上述的共享内存。
- 使用和宿主同样的 UID，保证文件权限不出错。

ROS 的 Docker 镜像默认使用 root 运行，对应容器外的 root，很可能会产生权限问题导致共享内存传输失败，必须修改 UID。修改 UID 有很多方法（[Running Docker Containers as Current Host User](https://jtreminio.com/blog/running-docker-containers-as-current-host-user/)），但我觉得直接`docker build`最简单。

这里还参考了 [rosblox/ros-template](https://github.com/rosblox/ros-template)，不过他的方法似乎有点问题。

首先，切换用户一步使用 ros2.dockerfile 实现

```dockerfile
FROM ros:humble

ARG ROS_UID
RUN adduser --disabled-password --uid $ROS_UID ros
USER ros
```

其中的 ROS_UID 需要指定，可以通过`id -u`查看。

使用 docker-compose.yml 管理上述选项

```yaml
services:
  ros2:
    build:
      context: ./
      dockerfile: ros2.dockerfile
      args:
        - ROS_UID=${ROS_UID}
    network_mode: host
    ipc: host
```

其中 ROS_UID 可以是环境变量，也可以是一个 [.env](https://docs.docker.com/compose/environment-variables/envvars-precedence/) 文件（一般第一个用户就是1000）

```bash
ROS_UID=1000
```

## 测试

在当前目录下创建上述几个文件后，分别在 Docker 内外以任意组合均可发布和接受均可正常收到，如
```bash
ros2 topic pub /test std_msgs/msg/String '{data: "hello"}'
docker compose run --rm -it ros2 ros2 topic echo /test
```
或是
```bash
docker compose run --rm -it ros2 ros2 topic pub /test std_msgs/msg/String '{data: "hello"}'
ros2 topic echo /test
```

## 其他方法

基于命名空间似乎也可以考虑：

<https://gist.github.com/heyfluke/b8372df866ec2584f9a51ca7d7fe9ebb>
<https://www.jujens.eu/posts/en/2017/Jul/02/docker-userns-remap/>
