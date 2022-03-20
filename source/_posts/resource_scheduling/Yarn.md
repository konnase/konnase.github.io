---
title: Yarn
p: resource_scheduling/Yarn
date: 2018-05-11 16:05:57
tags: [yarn, scheduling]
categories: Resource Management
typora-root-url: ../../../source
---

# Yarn
- 编程框架和资源管理器解耦
- 以container为单位调度资源
## ResourceManager
- 跟踪资源使用量和节点的活跃度
- 仲裁租户间的资源竞争
- 与客户端的接口，接受客户端提交的应用
- AMService，与AM通信--接受AM资源请求并资源container给AM
- 如果应用不合作了，RM可以让NM强制关掉应用所开启的所有containers
- RM崩溃并重启时会从持久化存储中加载其崩溃前存储的状态，并杀掉所有的容器并启动新的AM。正在尝试使RM重启时，AM可以继续运行

<!--more-->

## ApplicationMaster
- Yarn中AM没有直接和运行的container通信，而是通过RM来完成的。
  - NM将container信息报告给RM，RM将container信息发送给AM
- AM也可以指导NM杀掉container
- AM崩溃的话，不会持久化崩溃前的状态。故AM重启会使所有正在运行的或者已经完成的任务重新运行。

## NodeManager
- NM验证AM获得的lease（租约）的可用性后，配置容器的环境。需要复制所有需要的依赖到本地存储中
- NM一旦出错，运行在其上的任务都会被认为出错了。
  - NM重启时，会清掉所有本地状态
  - NM出错，AM负责处理该节点出错导致的任务出错问题。即将出错时运行在该NM上的任务重新运行一遍（重新向RM申请资源）。

## 应用程序生命周期
- 用户通过CLC（容器启动上下文）提交应用程序给RM
- RM指定NM运行ApplicationMaster，AM使用心跳向RM报告其活跃度和资源需求
- RM收到AM的资源请求，使用准入控制机制：根据应用需求、调度优先级和资源可用性向AM以container的形式提供资源。RequestResource包括：
  - container数目
  - 每个container的资源数目
  - 数据本地性偏好
  - 应用中请求的优先级
- AM获得资源，构造一个CLC在相关NM上启动拿到的container。AM可通过RM来监控container的状态
- AM运行完成后，向RM注销并干净的退出