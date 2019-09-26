---
title: Linux学习心得
p: system_management/Linux学习心得
date: 2017-08-05 15:56:40
tags:
categories: System
---


## Linux
### TTY
TeleTYpe的简称，用来新建一个终端来和计算机交互，TTY Driver会根据为每一个终端创建一个TTY设备

可以把TTY理解为一个管道，从一端写入的内容可以从另一端读出来

ssh可以远程连接计算机，但一旦这个连接断掉了，内核会给和该tty相关的进程发送SIGHUP信号，进程收到信号之后的默认操作是推出进程。这样一旦连接断掉，之前跑的东西就都停掉了，对于远程连接服务器跑实验来说无疑是不可接受的。tmux可以解决这个问题，tmux用来为ssh连接远程计算机。ssh客户端发送过来的数据包都会被tmux客户端接收到，然后tmux转发给tmux服务端，由服务端完成和远程计算机的交互

常用命令：
```bash
# creates a new tmux session named session_name
$ tmux new -s session_name
# attaches to an existing tmux session named session_name
$ tmux attach -t session_name
# switches to an existing session named session_name
$ tmux switch -t session_name
# lists existing tmux sessions
$ tmux list-sessions
# detach the currently attached session. ctrl-b + d means press key combination ctrl-b, then release the key combination, and then press key d
$ tmux detach (ctrl-b + d)
```

### 网络
linux刚安装系统如果上不了网，检查`/etc/sysconfig/network-scripts/`下面的网卡是否挂载，尝试`ifup eth0`，这里的eth0表示网卡名称