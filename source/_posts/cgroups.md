---
title: cgroups
date: 2018-05-13 14:00:30
tags: [cgroups]
---

# Cgroups
- 一个css_set关联多个 cgroups 层级结构的节点时，表明需要对当前css_set下的进程进行多种资源的控制。而一个 cgroups 节点关联多个css_set时，表明多个css_set下的进程列表受到同一份资源的相同限制。
- cgroups 也是通过 VFS 把功能暴露给用户态的，cgroups 与 VFS 之间的衔接部分称之为 cgroups 文件系统。VFS本身就是用 c 语言实现的一套面向对象的接口。
- Linux中，用户可以使用mount命令挂载 cgroups 文件系统