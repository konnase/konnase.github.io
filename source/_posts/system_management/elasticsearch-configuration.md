---
title: elasticsearch configuration
p: system_management/elasticsearch-configuration
date: 2018-05-30 13:43:48
tags: [elasticsearch]
typora-root-url: ../../../source
---

## DC上配置elasticsearch
### 配置node-master
- 在n125上，新建/etc/elasticsearch/elasticsearch.yml
```yaml
    cluster.name: elasticsearch_cluster
    node.name: node-master
    node.master: true
    node.data: true
    http.port: 9200
    transport.tcp.port: 9300
    transport.type: netty3
    http.type: netty3
    network.host: 0.0.0.0
    network.publish_host: esmaster.elsearch.crawl-mesos-project.menya.marathon.mesos
    discovery.zen.ping.unicast.hosts: ["esmaster.elsearch.crawl-mesos-project.menya.marathon.mesos:9300"]
    discovery.zen.ping_timeout: 3s
```
- 新建/data/crawl-es-data目录
- 执行`sysctl -w vm.max_map_count=262144`将机器的`vm.max_map_count`调高一些
- DC上创建service，id为`menya/crawl-mesos-project/elsearch/esmaster`，网络类型选择`VirtualNetwork`，指定镜像并根据官方镜像说明，传入一些环境变量
- 运行service

<!--more-->

### 配置node-data
- 在n138上，新建/etc/elasticsearch/elasticsearch.yml
```yaml
    cluster.name: elasticsearch_cluster
    node.name: node-data-1
    node.master: false
    node.data: true
    http.port: 9200
    transport.tcp.port: 9300
    transport.type: netty3
    http.type: netty3
    network.host: 0.0.0.0
    network.publish_host: esnode1.elsearch.crawl-mesos-project.menya.marathon.mesos
    discovery.zen.ping.unicast.hosts: ["esmaster.elsearch.crawl-mesos-project.menya.marathon.mesos:9300"]
    discovery.zen.ping_timeout: 3s
```
- 新建/data/crawl-es-data目录
- 执行`sysctl -w vm.max_map_count=262144`将机器的`vm.max_map_count`调高一些
- DC上创建service，id为`menya/crawl-mesos-project/elsearch/esnode1`，网络类型选择`VirtualNetwork`，指定镜像并根据官方镜像说明，传入一些环境变量
- 运行service
- 同理，在n126上创建node-data-2。

如此，就会创建一个包含node-master、node-data-1和node-data-2的elasticsearch集群。

### 配置
n125--esmaster
n138--esnode1
n126--esnode2


### 坑
- `sysctl -w vm.max_map_count=262144`一定要在elasticsearch容器运行的宿主机上执行，不然会报错说`vm.max_map_count`太小
- 如果一直dataNode节点一直连接不上node-master
  - 首先看dataNode节点解析的`esmaster.elsearch.crawl-mesos-project.menya.marathon.mesos`地址对不对，如果不对可能是DNS缓存的问题，重启一下dataNode
  - 如果地址解析正确，则有可能是超时响应，提示`time out`，则将超时时间设置长一点，如设置为3s：`discovery.zen.ping_timeout: 3s`