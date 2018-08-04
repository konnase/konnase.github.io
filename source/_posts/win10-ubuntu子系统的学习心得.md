---
title: win10 ubuntu子系统的学习心得
date: 2017-07-23 15:08:48
tags: [ubuntu in win10,configuraton]
categories: System
---

## Ubuntu子系统的文件存放位置

> 网上搜索说在C:\Users\lqp19\AppData\Local\lxss目录下面，但本机上没有这个目录
<!--more-->
![image](/img/p1.PNG)

## 配置文件

> 每次进入Ubuntu子系统，都需要将/etc/profile文件中的配置重新生效一次，否则配置的诸如JAVA_HOME、TOMACAT_HOME等环境变量都不能识别，而且执行完：source /etc/profile 之后，terminal中的字体颜色也会随着变化 
> 

## /etc/hosts文件
> 修改/etc/hosts文件过后，保存退出vim编辑器，此时可以使用刚编辑过的配置，但是一旦关闭命令行窗口，重新进入Ubuntu子系统，之前保存的/etc/hosts文件里面的内容就没有了。