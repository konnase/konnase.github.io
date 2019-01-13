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