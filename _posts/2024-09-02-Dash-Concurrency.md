---
layout: post
title: Dash Shell 并行编程
date: 2024-09-02 00:22:56 +0800
tags: 
- 编程
render_with_liquid: false
---

在`dash`中，我们经常需要并行运行一些程序，此时通常可以使用`&`并行运行程序，但脚本通常不会等待子进程结束就退出，因此，可以使用下列模板：

```shell
# trap '' INT
trap wait INT
ping baidu.com &
ping bing.com &
wait
echo "All subprocesses exited"
```

注意为了在收到 SIGINT 后等待子进程退出，在开始时额外使用`trap wait INT`指令，即脚本本身在收到 SIGINT 后不退出，等待子进程结束。也可以写`trap '' INT`，直接忽略 SIGINT，等待各进程自行退出。

注意 SIGINT 默认发给整个前台进程组，即包括子进程和主进程，因此无需主动使用`kill -INT $pids`发送 SIGINT。如果使用，还可能因为进程退出太快产生警告。参考 [Behavior of SIGINT with Bash](https://superuser.com/a/1830133)。

如果需要具体子进程的 PID，则可以用下列模板

```shell
trap '' INT
ping baidu.com &
pids="$pids $!"
ping bing.com &
pids="$pids $!"
wait $pids
echo "All subprocesses exited"
```

其中`$!`就是上一个后台进程的 PID，用`pids`变量记录所有进程的 PID。
