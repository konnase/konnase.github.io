---
title: Mxnet
p: deep_learning/Mxnet
date: 2018-09-16 19:02:01
tags: [mxnet]
categories: Machine Learning
---

# Mxnet训练流程
- 定义好symbol：符号式编程，生成描述计算的计算图，Symbol API 这个包主要是用于提供神经网络的图和自动求导。使用c++写好的Symbol类，可进行加减乘除等符号运算。

- 写好dataiter：如果是分布式训练，需要将训练数据平均切分到每台训练的机器上

- 初始化参数：cpu or gpu、学习率、优化器参数、针对不同网络设置参数初始化方法、验证度量(evaluation metrics)、loss function、callbacks

- 模型训练：
  - 创建Module: 
  ```python
  mod = mx.mod.Module(symbol=net,
      context=mx.cpu(),
      data_names=['data'],
      label_names=['softmax_label'])
  ```
  - mod.fit():
    - mod.bind(): 分配内存，为计算做准备
    - mod.inst_prams(): 初始化module参数
    - mod.init_optimizer(): 初始化优化器，默认sgd
    - mod.metri.create(): 创建evaluation metric
    - mod.forward(): 前向传播计算
    - mod.update_metric(): 更新预测精度
    - mod.backward(): 反向传播，更新计算梯度
    - mod.update(): 更新参数

<!--more-->

## Mxnet参数服务器架构
- server节点可以跟其他server节点通信，每个server负责自己分到的参数，server group共同维持所有参数的更新
- worker节点之间没有通信，只跟**自己对应的server**进行通信
- 每个worker group有一个task scheduler，负责向worker分配任务，并且监控worker的运行情况。当有新的worker加入或者退出，task scheduler 负责重新分配任务。

## 参考文献
[MXnet之ps-lite及parameter server原理](https://www.cnblogs.com/heguanyou/p/7868596.html)