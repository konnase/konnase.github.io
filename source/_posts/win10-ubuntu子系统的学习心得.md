---
title: win10 ubuntu子系统的学习心得
date: 2017-07-23 15:08:48
tags: [ubuntu,win10]
---

## Ubuntu子系统的文件存放位置

> 网上搜索说在C:\Users\lqp19\AppData\Local\lxss目录下面，但本机上没有这个目录
<!--more-->
![image](/img/p1.PNG)

## 配置文件

> 每次进入Ubuntu子系统，都需要将/etc/profile文件中的配置重新生效一次，否则配置的诸如JAVA_HOME、TOMACAT_HOME等环境变量都不能识别，而且执行完：source /etc/profile 之后，terminal中的字体颜色也会随着变化 