---
title: Tensorflow
p: deep_learning/Tensorflow
date: 2018-12-20 10:48:39
tags: [deep learnig, tensorflow]
mathjax: true
---
<!-- TOC -->

- [1. 模型](#1-%E6%A8%A1%E5%9E%8B)
    - [1.1. Graph and Session](#11-graph-and-session)
        - [1.1.1. 概念](#111-%E6%A6%82%E5%BF%B5)
    - [1.2. 数据并行](#12-%E6%95%B0%E6%8D%AE%E5%B9%B6%E8%A1%8C)
        - [1.2.1. in-graph replication](#121-in-graph-replication)
        - [1.2.2. between-graph replication](#122-between-graph-replication)

<!-- /TOC -->
# 1. 模型
## 1.1. Graph and Session
在分布式参数服务器架构训练中，`tf.train.replica_device_setter(ps_tasks=3)`可以用来指定将tf.Variable的放置位置，比如：
```python
with tf.device(tf.train.replica_device_setter(ps_tasks=3)):
  # tf.Variable objects are, by default, placed on tasks in "/job:ps" in a
  # round-robin fashion. And only Variable ops are placed on ps tasks
  w_0 = tf.Variable(...)  # placed on "/job:ps/task:0"
  b_0 = tf.Variable(...)  # placed on "/job:ps/task:1"
  w_1 = tf.Variable(...)  # placed on "/job:ps/task:2"
  b_1 = tf.Variable(...)  # placed on "/job:ps/task:0"

  input_data = tf.placeholder(tf.float32)     # placed on "/job:worker"
  layer_0 = tf.matmul(input_data, w_0) + b_0  # placed on "/job:worker"
  layer_1 = tf.matmul(layer_0, w_1) + b_1     # placed on "/job:worker"

# or place worker
with tf.device(tf.train.replica_device_setter( 
                       worker_device='/job:worker/task:'+task_idx, 
                       clustercluster=cluster)):
        # build your model here as if you only were using a single machine
        with tf.Session(server.target):
            # train your model here 
```
tf.Variable 对象默认是放在ps上的，也只有Varibale是放在ps上的

ps上面只是存储了模型的参数，多个ps会将模型的参数拆分之后分别进行存储，workers计算参数更新之后，会将相应的参数发送到对应的ps上去。

`tf.train.replica_device_setter`会返回一个device function，用于`with tf.device(device_function)`：在Operation对象构造时自动给Operation对象

<!--more-->

### 1.1.1. 概念
- client：编写代码的程序，client里面构造了tf.Graph，并构建tf.Session来启动训练
- tf.Session：Tensorflow使用tf.Session类来表示client程序（通常为Python）与c++运行时之间的连接。
```python
# Create a default in-process session.
with tf.Session() as sess:
    # ...

# Create a remote session.
with tf.Session("grpc://example.org:2222"):
    # ...
```
- cluster：对于一个cluster，由tf.train.ClusterSpec指定，里面包含了一些Server，Server又可以分为两种job（ps和worker），每个job可以有多个tasks

## 1.2. 数据并行
数据并行分为图内复制和图间复制，
### 1.2.1. in-graph replication
图内复制中所有的Operation都在一张tf.Graph中；用一个客户端来生成Graph， 把所有tf.Operation分配到所有的ps和worker上。一般是单机多卡的训练。
### 1.2.2. between-graph replication
每个/job/worker的task都有独立的client，参数通过tf.trian.replica_device_setter复制到task中，而每个worker上都会有模型计算部分的副本

框架  代码过程 
参数更新的时候的通信