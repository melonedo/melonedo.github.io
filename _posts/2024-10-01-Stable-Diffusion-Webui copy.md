---
layout: post
title: Stable Diffusion web UI 安装
date: 2024-10-01 20:27:00 +0800
tags: 
- 编程
render_with_liquid: false
---

大体上步骤是下载 https://github.com/AUTOMATIC1111/stable-diffusion-webui，然后运行 webui.sh，在浏览器的 localhost:7860 就可以看到结果了。

# Python 版本
由于 numpy 版本限制，python 必须使用 3.10 （3.11 可能也可以）。在 Ubuntu  20.04 下参考：[Unable to install webui in ubuntu 20.04, python version is 3.8.10. TypeError: 'type' object is not subscriptable. #13976](https://github.com/AUTOMATIC1111/stable-diffusion-webui/discussions/13976)。其中`python_cmd`在 webui-user.sh 设置。

> Python 3.10 can be installed from the [deadsnakes](https://launchpad.net/~deadsnakes/+archive/ubuntu/ppa) PPA.
> 
> ```
> sudo add-apt-repository ppa:deadsnakes/ppa
> sudo apt install python3.10
> ```
> Then set [`python_cmd`](https://github.com/AUTOMATIC1111/stable-diffusion-webui/blob/5ef669de080814067961f28357256e8fe27544f4/webui-user.sh#L16) to `python3.10`. Delete the venv folder if it exists.
> 
> Or you can install it with an alternative package manager such as [conda](https://docs.conda.io/en/latest/).

# xformers

```
no module 'xformers'. Processing without
```

这个消息是说可以使用`bash webui.sh --xformers`的形式运行，这样会使用 xformers 这个库。
[no module 'xformers'. Processing without .. UGH #15939](https://github.com/AUTOMATIC1111/stable-diffusion-webui/discussions/15939)

# 代理

```
ValueError: When localhost is not accessible, a shareable link must be created. Please set share=True
```

代理会导致服务器开启失败，总之先删掉环境变量吧。

[[Bug]: VPN cause "ValueError: When localhost is not accessible, a shareable link must be created. Please set share=True." #9089](https://github.com/AUTOMATIC1111/stable-diffusion-webui/issues/9089)

# 界面左下角是«↑ ↓ Viewing <>

这是进了`w3m`，q 键退出。
