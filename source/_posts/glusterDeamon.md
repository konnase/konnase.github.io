---
title: glusterDeamon
date: 2017-07-07 15:50:48
tags: [glusterfs,install]
categories: System
---

## connection failed check if gluster deamon is running

在使用gluster命令的时候，如果之前在本机上创建的volume所在的局域网ip与现在本机的局域网ip不是同一个的话，就会经常出现gluster deamon不可用的情况，网上搜索了很多方法，感觉最实用的还是重装glusterfs，当然前提是gluster上的数据不那么重要。。。

> 执行：

    cd /var/log/glusterfs/
    rm -fR *
    rm -rf /var/lib/glusterd/*

> 然后直接卸载重装：

	sudo apt-get remove glusterfs-common
	sudo add-apt-repository ppa:gluster/glusterfs-3.8
	sudo apt-get update
	sudo apt-get install glusterfs-server

## mount 出错
查看/var/log/glusterfs/export.log出现：failed to get the port number for remote subvolume. Please run 'gluster volume status' on server to see if brick process is running
执行`gluster volume status`发现本机Brick的TCP Port没有分配，即volume没有启起来，执行`gluster volume vol-name start force`即可
* * *

