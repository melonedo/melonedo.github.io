---
layout: post
title: 火狐标签页丢失抢救
date: 2025-06-17 10:55:00 +0800
tags:
- 网络
render_with_liquid: false
---

第一步：先别急着重启浏览器（大概率重启也没用），先在地址栏进入`about:support`，在里面找到 profile folder 里面的 sessionstore-backups 文件夹，复制到别的地方备份。

第二步：这个文件夹里面都是标签页记录的备份，包括上次的，上上次的，以及每次更新时的。可以用 [Session History Scrounger for Firefox](https://www.jeffersonscher.com/ffu/scrounger.html) 查看内容，挑一个看里面的内容，有可能只是标签页的格式损坏，内容可能还在，可以做记录保留。

第三步：关闭火狐，从上面的文件夹里面挑一个文件，重命名为sessionstore.jsonlz4，替换掉 profile folder 里的同名文件（这个文件只在火狐关着的时候有），重新启动。一个不行就换别的更早的。


参考：
- [Lost tabs, no history](https://support.mozilla.org/en-US/questions/1407584)
- [Lost my entire session of over 100 tabs! sessionstore.js has the tabs though? ](https://www.reddit.com/r/firefox/comments/78i49o/lost_my_entire_session_of_over_100_tabs/)
