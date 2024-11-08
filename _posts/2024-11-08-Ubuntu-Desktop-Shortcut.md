---
layout: post
title: Ubuntu 下创建桌面快捷方式
date: 2024-11-06 11:08:00 +0800
tags: 
- 编程
render_with_liquid: false
---

在 Linux 下我们经常会写一堆的脚本用于运行命令，但是经常要输入命令很不方便，那么能不能像 Windows 一样在桌面建快捷方式呢？答案是可以而且非常简单。

我们只需要在`~/Desktop`下新建一个`app.desktop`文件，内容是
```
[Desktop Entry]
Type=Application
Name=<name>
Exec=gnome-terminal --hide-menubar -t <name> -- <command>
Path=<pwd>
```
此时，在桌面上就会看到一个 app.desktop 图标，不过还无法运行。如果要运行，可以右键单击在菜单中点击允许运行，或者[以命令的形式](https://askubuntu.com/a/1391903)：
```shell
gio set ~/Desktop/app.desktop metadata::trusted true
chmod a+x ~/Desktop/app.desktop
```
注意，如果程序运行失败，可能不会有报错，可以修改命令保证运行成功来查看日志。
