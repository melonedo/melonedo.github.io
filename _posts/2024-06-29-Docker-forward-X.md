---
layout: post
title: Docker 显示图形界面
date: 2024-06-09 10:42:00 +0800
tags: 
- 编程
render_with_liquid: false
---

在 Docker 里显示图形界面这事涉及很多层次的问题，这里记录一下：

## 基本操作

为了简化，这里使用`gns3/xeyes`镜像，这个镜像会在容器中运行`xeyes`命令。基本的运行指令是

```
docker run --rm gns3/xeyes
```

这样肯定会失败，提示`Error: Can't open display:`，因为容器中默认是没有图像输出的。

## 配置参数

首先需要配置`DISPLAY`环境变量，并且由于使用的是转发的 X11，使用主机的网络，并且加载 .Xauthority 用于管理权限。

```
docker run --rm --net=host -e DISPLAY=$DISPLAY -v $HOME/.Xauthority:/root/.Xauthority:ro gns3/xeyes
```

这样就能看到 xeyes 的界面了。另外还有别的配置方法，这里简单记录一下：

```
sudo xhost +local:docker
docker run --rm -v /tmp/.X11-unix:/tmp/.X11-unix --env DISPLAY=$DISPLAY --net=host -v "$HOME/.Xauthority:/root/.Xauthority:rw"  -v $PWD:/scripts gns3/xeyes
```

另外，在 ROS 的教程里对此有很详细的介绍：<http://wiki.ros.org/docker/Tutorials>。
