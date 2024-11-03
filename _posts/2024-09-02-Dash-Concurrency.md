---
layout: post
title: Dash Shell 并行编程
date: 2024-09-02 22:56:00 +0800
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

如果怕没有执行到最后（一般是由于`set -e`），可以`trap wait EXIT`，总是等待所有进程退出。注意最好不要`trap wait EXIT INT`，这样会在退出后直接进`wait`，收到 SIGINT 后异常退出，不再等待。

实践表明很多程序可能 SIGINT 不太使用，可以转为发送 SIGTERM，即在脚本开头写：
```shell
trap wait EXIT # 必须等待子进程退出
trap 'kill -- -$$' INT # 向子进程发送 SIGTERM（kill 默认）
trap '' TERM # 别把自己停了
```

注意 SIGINT 默认发给整个前台进程组，即包括子进程和主进程，因此无需主动使用`kill -INT $pids`发送 SIGINT。如果使用，还可能因为进程退出太快产生警告。参考 [Behavior of SIGINT with Bash](https://superuser.com/a/1830133)。

如果需要在退出时向子进程发信号，但没有记录子进程的PID（`$!`），可以使用`kill -INT -$(echo $(ps -p$$ o tpgid=))`向当前 tty 所在的所有进程发送 SIGINT，模拟 ctrl+c。

如果需要具体子进程的 PID，则可以用下列模板

```shell
trap '' INT
trap 'wait $pids && echo "All subprocesses exited"' EXIT
ping baidu.com &
pids="$pids $!"
ping bing.com &
pids="$pids $!"
```

其中`$!`就是上一个后台进程的 PID，用`pids`变量记录所有进程的 PID。

开的进程多了之后，如果要在行前加上区分每个进程的符号，可以使用`sed`，参考：<https://stackoverflow.com/questions/42482477/add-prefix-in-bash-command-output>
```shell
trap '' INT
ping baidu.com 2>&1 | sed -u "s/^/prefix1> /" &
ping bing.com 2>&1 | sed -u "s/^/prefix2> /" &
wait
echo "All subprocesses exited"
```
这样输出的结果就会带上前缀了。不过要注意，这样有时可能会导致结果攒到最后一起输出，`stdbuf`似乎也没用。