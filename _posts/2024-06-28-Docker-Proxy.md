---
layout: post
title: Docker 设置代理
date: 2024-06-28 22:36:00 +0800
tags: 
- 编程
render_with_liquid: false
---

随着 Docker 在国内访问愈加困难，设置代理也是非常必要的了。

## 关于代理

Docker 中代理分为两类，一类是 Docker 容器里运行的系统的代理，一类是 Dockerd 本身所使用的代理。这里配置的是 Dockerd 本身的代理。

## 配置方法

Dockerd 设置代理有三种设置方法：[Configure the daemon to use a proxy](https://docs.docker.com/config/daemon/proxy/)

- 在运行时添加`--http-proxy`
- 在运行时使用环境变量`HTTP_PROXY`
- 在[daemon.json](https://docs.docker.com/config/daemon/#configuration-file)中添加"http-proxy"参数。

但是，由于大部分人的 dockerd 应该是通过 systemd 管理的，因此实际上管理的方法有两种：

- 修改/etc/docker/daemon.json。

注意这里的`https-proxy`一般和`http-proxy`一致，而 Docker 官方的示例里前缀是`https://`，不适合大多数本地的代理软件。

```json
{
  "proxies": {
    "http-proxy": "http://localhost:7890",
    "https-proxy": "http://localhost:7890",
    "no-proxy": "*.test.example.com,.example.org,127.0.0.0/8"
  }
}
```

修改完后需要使用`sudo systemctl restart docker`重启 dockerd。

- 新建 systemd 服务文件`/etc/systemd/system/docker.service.d/http-proxy.conf`

```
[Service]
Environment="HTTP_PROXY=http://localhost:7890"
Environment="HTTPS_PROXY=http://localhost:7890"
```

执行命令重新加载 systemd 配置文件并重启 dockerd：

```
sudo systemctl daemon-reload
sudo systemctl restart docker
```

执行`docker info`命令如果出现`HTTP Proxy`相关内容说明配置成功。

## 调试

如果上述 daemon.json 配置错误，[`sudo systemd status docker`很可能不会给出有用的信息](https://stackoverflow.com/questions/39100641/docker-service-start-failed)。

```
⋊> mlinux@mlinux ⋊> ~/Code systemctl restart docker
Failed to start docker.service: Interactive authentication required.
See system logs and 'systemctl status docker.service' for details.
⋊> mlinux@mlinux ⋊> ~/Code sudo systemctl status docker.service
● docker.service - Docker Application Container Engine
     Loaded: loaded (/lib/systemd/system/docker.service; enabled; vendor preset: enabled)
     Active: failed (Result: exit-code) since Fri 2024-06-28 22:28:56 CST; 1min 41s ago
TriggeredBy: ● docker.socket
       Docs: https://docs.docker.com
    Process: 2311811 ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock (code=exited, sta>
   Main PID: 2311811 (code=exited, status=1/FAILURE)

6月 28 22:28:56 mlinux systemd[1]: Stopped Docker Application Container Engine.
6月 28 22:28:56 mlinux systemd[1]: docker.service: Start request repeated too quickly.
6月 28 22:28:56 mlinux systemd[1]: docker.service: Failed with result 'exit-code'.
6月 28 22:28:56 mlinux systemd[1]: Failed to start Docker Application Container Engine.
```

这时候直接执行`dockerd`命令，可以得到报错：
```
⋊> mlinux@mlinux ⋊> ~/Code sudo dockerd
unable to configure the Docker daemon with file /etc/docker/daemon.json: the following directives don't match any configuration option: default
```
