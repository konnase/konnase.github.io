---
title: cgroups
p: resource_scheduling/cgroups
date: 2018-05-13 14:00:30
tags: [cgroups]
categories: Resource Management
---

# Cgroups
安装libcgroup （yum install libcgroup）
- 一个css_set关联多个 cgroups 层级结构的节点时，表明需要对当前css_set下的进程进行多种资源的控制。而一个 cgroups 节点关联多个css_set时，表明多个css_set下的进程列表受到同一份资源的相同限制。
- cgroups 也是通过 VFS 把功能暴露给用户态的，cgroups 与 VFS 之间的衔接部分称之为 cgroups 文件系统。VFS本身就是用 c 语言实现的一套面向对象的接口。
- Linux中，用户可以使用mount命令挂载 cgroups 文件系统

<!--more-->

## cpu子系统和cpuacct子系统
- cpu子系统用来做cpu资源隔离，而cpuacct子系统是用来做cpu资源统计的
- cpu子系统通过cpu.shares来保证任务最少可以使用的cpu资源数目
- 在cgroup挂载目录/sys/fs/cgroup/cpu里面新建一个目录比如test，则test会被自动加入到cpu子系统中去
### 让job运行在cgroup中
安装libcgroup-tools （yum install libcgroup-tools），使用cgexec执行hello.sh脚本如下：
```bash
#!/bin/bash
i=0
while(( $i<10000 ))
do
  echo "hello $i\n"
  let "i++"
  sleep 1
done
```
执行`cgexec -g cpu:test ./hello.sh`，则hello.sh将运行在test cpu子系统中，进入/sys/fs/cgroup/test查看tasks可以看到hello.sh脚本运行时对应的pid

### CPU子系统字段
- cpu.cfs_period_us 为cfs调度器的一个时间周期的长度，默认为100000，即100000ns=100ms
- cpu.cfs_quota_us 为只能使用0.5个cpu
- cpuacct.stat 报告cgroup的所有任务（包括其子孙层级中的所有任务）使用的用户和系统CPU时间。user和system代表用户态和内核态，单位为USER_HZ

### 如何计算容器CPU利用率
Linux Kernel 将 CPU 使用的信息记录在了 /proc/stat 这个文件中，该文件中记录了，如下所示：
```bash
[root@sean ~]# cat /proc/stat
cpu  1344357 729 298522 107841813 41774 0 1754 0 0 0
cpu0 1344357 729 298522 107841813 41774 0 1754 0 0 0
intr 206534153 29 10 0 0 154 .......
ctxt 317949799
btime 1524823711
processes 677986
procs_running 3
procs_blocked 0
softirq 66031632 2 35326999 1 2928073 535817 0 32 0 0 27240708
```
该文件记录了所有进程花费的总的cpu时间，第一行所有项（user、nice、system、idle等）之和就是所有cpu从开机到现在的总CPU时间Tmachine。单位是纳秒

容器使用的CPU时间Tcontainer可以从 cpuacct/cpuacct.usage 中获得，单位是纳秒；也可以从`cpuacct/stat`中获得，包括user和system，单位是 `USER_HZ`，一般为100，故`cpuacct/stat`中的值一般乘以`1/100`就是秒为单位。

则容器的cpu利用率为：`(Tcontainer - prevTcontainer) / (Tmachine - prevTmachine)`