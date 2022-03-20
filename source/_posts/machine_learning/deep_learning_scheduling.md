---
title: 分布式深度学习资源调度
p: deep_learning/deep_learning_scheduling
date: 2018-09-14 15:48:39
tags: [deep learnig, scheduling]
categories: Machine Learning
mathjax: true
typora-root-url: ../../../source
---

## 分布式机器学习相关会议
- 各大机器学习会议上（NIPS， ICML，ICLR）
- 各大系统会议上（SOSP，OSDI，ATC，EuroSys，SoCC）
- 应用对应的顶级会议上（CVPR，KDD）

## 常用的深度学习数据集
- ImageNet：15 million 带标签的高分辨率224*224的RGB图像，共22,000catagories，1.2TB
- ImageNet 2012: 1.2 million 带标签的高分辨率224*224的RGB图像，共1,000个分类，138GB；50,000张测试图像，1,000个分类
- MNIST：输入是28*28是的二值图，输出是0-9这是个数字，60,000 训练图像，10,000测试图像
- Cifar-10：由60,000张32*32的RGB彩色图片构成，共10个分类。50,000张训练，10,000张测试（交叉验证）

## 现有的一些分布式深度学习平台
- 腾讯的Mariana
- Google的DistBelif
- 微软的Adam

<!--more-->

## 分布式机器学习的研究领域

### 从统计、优化理论、优化算法角度来做

关注以下问题：通过分布式并行或其他方法加速训练后，这个新算法还能保证收敛到之前相同的最优值（或者一个满意的最优值）吗？在分布式环境下，收敛能有多快，比非分布式训练快多少？收敛的有多接近，和单机上跑出来的最优解一样吗？收敛需要什么假设？*应该怎么设计训练过程*（比如，怎么抽样数据、怎么更新参数）从而保证能接近某个最优解，同时还保证加速？

### 从机器学习的模型角度

修改原有的模型；提出新的模型；

### 优化方向
- 异步训练：参数服务器架构；减小通信开销（计算开销<<通信开销）
- 数据并行：将数据分成很多小块，加快训练模型；要求数据之间没有依赖性
- 模型并行：如何划分模型？模型之间的关联性要考虑

  Petuum和Tensorflow既支持数据并行也支持模型并行
  - 将CNN中的不同层放到不同的device(CPU or GPU)上
    
    这样拆分使得参数被分配到多个device上，对参数量巨大的模型就可以提供很好的支持
  - 将CNN中某一层拆开放到不同的device上
    
    实现难度较大

- 对反向传播算法的重调度：使计算时间和通信时间尽可能重叠
- 模型更新：对于全连接层，反向传播的时候，下一层计算的$\Delta w$

### 并行收敛算法满足的假设
- 训练程序的计算任务集中在参数更新函数上
- 每个iteration，数据之间没有依赖性
- 每个iteration开始之前，每个计算节点需要比较容易的拿到模型参数；每个节点计算完成之后，需要比较容易的将梯度收集起来应用于参数更新:
$ \theta(t+1) = \theta(t) - \alpha * \Delta(\theta(t),D_p(t)) $

前两个条件容易满足，但第三个条件在多机环境下涉及网络通讯，如何保证参数共享以及梯度同步成了保证第三个条件要解决的核心问题。

### All reduce并行算法
设数据量为 X，N GPUs，如果使用一个main gpu来做参数更新并把更新之后的参数再发送会其他worker，则此过程交换的参数为`2*X*(N-1)`

这是 Baidu Silicon Valley AI Lab在GTC 2017上提出了allreduce的实现算法，一些分析如下：

#### Multi-Tower Scaling
[Effectively scaling deep learning framework](http://on-demand.gputechconf.com/gtc/2017/presentation/s7543-andrew-gibiansky-effectively-scakukbg-deep-learning-frameworks.pdf)提到分布式训练中网络开销巨大，计算节点之间传递的参数数目太多，如果网络带宽不大，传输开销很大，影响扩展性。

下图指出使用一个main gpu来做参数同步的话，在他们的benchmark中，使用Titan X Maxwell GPU计算单个gpu发来的参数耗时在300ms左右，如果N的数目巨大，这将成为巨大的瓶颈。同时，通信开销在(N-1)/3 s，表明通信开销也是随N的数目线性增长。
![communication overhead](/img/communication_overhead.png)
下图指出随着N的数目的增大，花在通信上的开销的比列越来越大
![communication overhead](/img/communication_overhead_1.png)
遂提出了ring all reduce算法来解决高communication overhead的问题

##### 采用allreduce方式后

  ![allreduce](/img/pscl_fig_2_allreduce.png)
  ![allreduce](/img/pscl_fig_4_allreduce.png)
  ![allreduce](/img/pscl_fig_5_allreduce.png)

  Process 1-4上面分别由4个chunk
  - scatter reduce: 执行完N-1步之后，每个Process上都会有一个完整的chunk（比如Process 4上包含了c\_11, c\_21, c\_31和c\_41，即所有进程的chunk 1）
  - all gather: 然后将完整的chunk overwrite到其他不含有完整chunk的Process上

  每个GPU会迭代2(N-1)次，并在每次迭代中发送 X/N 大小的数据。则完成一次reduce过程交换的参数量为`2*X*(N-1)/N`

##### Scaling with tensorflow
  下图指出，Baidu在tensorflow的训练流图中的back prop 和weight update之间增加了ring allreduce，使用MPI实现GPUs之间的参数传递

  ![scaling with tensorflow](/img/scaling_with_tensorflow.png)

  结果是使用40个GPUs，训练速度几乎呈线性增长

## 分布式深度学习调度
### 多任务调度
深度学习作业都是运行时间较长的作业，所以一般一个timeslot都会比较大，比如可能40 min或者1 hour

### 深度学习集群的异构性
每个深度学习集群包括多个infiniband区域，每个infiniband又由多个机架（rack）组成。不同的机架上可能会安装有不同的加速设备（比如GPU、TPU、FPGA等）。这些加速设备可能会有一个PCIe接口，更进一步，可能会使用不同供应商特制的连接技术进行交互。这些设备或是直接与CPU连接，或是通过PCIe交换机与CPU连接。
资源管理系统使用configuration set来描述设备类型、一个server上的设备拓扑和一个infiniband区域内的网络拓扑，scheduler使用bitmap获取设备资源用量以及表达资源请求量。

#### 单个训练作业的性能
单个job，如果拥有多个GPU，将job打包的越紧密，训练效果会越好。不过有些模型（如inception V3）可以忍受较为松散的packing

#### 多个训练作业间的干扰
深度学习作业依赖于GPU做计算加速，但是它还是要通过PCIe总线与CPU频繁的通信。因此，在同一server上的多个jobs之间会存在干扰。不同jobs放置的越近，干扰越大，则性能下降得更多
![inter_job_inference](/img/inter_job_inference.png)

#### 解决方案
job scheduler利用资源抽象视图，在调度的时候考虑数据本地性和设备拓扑结构。集群空闲时，将jobs分散到集群中来避免干扰；集群高负荷时，将jobs pack得更紧凑，来给多GPU的jobs腾出空间

### CPU的重要性
在GPU训练中，CPU也扮演者重要的角色，下图说明CPU对训练性能的影响
![cpu_demand_to_reach_max_perf](/img/cpu_demand_to_reach_max_perf.png)

像Tensorflow会将GPU上的计算和CPU上的数据处理overlap起来，这样使得：如果GPU分配的好，即计算很快，那么CPU的用量将会很高

## 参考文献
- [张昊的知乎专栏](https://zhuanlan.zhihu.com/p/30976469)
- [technologies behind distributed deep learning allreduce]distributed(https://preferredresearch.jp/2018/07/10/technologies-behind--deep-learning-allreduce/)
- [All you need to know about scheduling deep learning jobs](https://www.sigops.org/src/srcsosp2017/sosp17src-final35.pdf)
- [Scheduling CPU for GPU-based Deep Learning Jobs](https://dl.acm.org/citation.cfm?id=3275445)