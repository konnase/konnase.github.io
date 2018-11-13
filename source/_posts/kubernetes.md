---
title: kubernetes
date: 2018-09-10 20:26:35
tags: [kubernetes]
---

# 设计架构
## Scheduler
k8s的有两步调度策略
- 预选策略(predicates)：强制性规则，遍历所有Node，选择满足要求的Node列表。v1.7支持15种策略
  - PodFitsResources：检查主机资源是否满足pod需求
  - NoDiskConfilict：检查主机上的卷冲突
  - MatchInterPodAffinity：检查pod和其他pod是否符合亲和性规则
- 优选策略(Priorities)：在预选策略的基础上，给Node打分排序，选出最优者。v1.7支持10个策略，每项策略对应一个权重，每项策略的得分乘以权重得到总分
  - ImageLocalityPriority：根据主机上是否已具备Pod运行的环境来打分，得分计算：不存在所需镜像，返回0分，存在镜像，镜像越大得分越高
  - LeastRequestedPriority：计算Pods需要的CPU和内存在当前节点可用资源的百分比，具有最小百分比的节点就是最优，得分计算公式：cpu((capacity – sum(requested)) * 10 / capacity) + memory((capacity – sum(requested)) * 10 / capacity) / 2

  <!--more-->

## Serviceaccount
一个角色拥有指定的资源权限（可以访问k8s里面的哪些资源，如pods、sercices等）和功能权限（CRUD），通过将RoleBinding将一个role绑定到某个serviceaccount上，即可让资源可通过serviceaccount来访问。关系图如下：
![image](/img/resource--role.png)