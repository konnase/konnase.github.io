---
title: 部署DC环境
date: 2018-03-29 15:12:49
tags: [docker, mesos, marathon, zookeeper, glusterfs, calico, docker registry]
---

# 基础环境搭建
四台阿里云集群：
```
172.0.0.1    master1
172.0.0.2    agent1
172.0.0.3    agent2
172.0.0.4    agent3
```
其中172.0.0.1机器可以访问外网(47.0.0.1)。
<!--more-->

## 安装Docker
每台机器都安装
- 安装必要的packages
  ```bash
  sudo yum install -y yum-utils \
    device-mapper-persistent-data \
    lvm2
  ```
- 配置阿里镜像源
  ```bash
  sudo yum-config-manager \
      --add-repo \
      http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
  ```
- 安装docker
  ```bash
  yum install docker-ce
  ```
- 开启docker服务
  ```bash
  systemctl start docker
  ```
### 遇到的问题：
  docker daemon不能启动，即执行`systemctl start docker`时提示`Job for docker.service failed because the control process exited with error code. See "systemctl status docker.service" and "journalctl -xe" for details. `，执行`systemctl status docker.service `，输出` Active: failed (Result: start-limit)  Process: 7880 ExecStart=/usr/bin/dockerd (code=exited, status=1/FAILURE) Main PID: 7880 (code=exited, status=1/FAILURE)`。
  解决方法：
  ```bash
  rm -rf /var/lib/docker/
  # 添加如下内容
  # vim /etc/docker/daemon.json
  {
      "graph": "/mnt/docker-data",
      "storage-driver": "overlay"
  }
  ```

## 安装Glusterfs
- 每台机器都安装
  ```bash
  yum install glusterfs
  yum install glusterfs-cli
  yum install glusterfs-libs
  yum install glusterfs-server
  yum install glusterfs-fuse
  yum install glusterfs-geo-replication
  ```
- 在每台机器上的/etc/hosts文件中添加hosts信息：
  ```bash
  172.0.0.1    master1
  172.0.0.2    agent1
  172.0.0.3    agent2
  172.0.0.4    agent3
  ```
- 加入peer
  ```bash
  # master1上：
  gluster peer probe agent1
  gluster peer probe agent2
  gluster peer probe agent3
  # 在各台机器上查看peer status
  gluster peer status
  ```
- 创建两个volume，一个做复制卷(replica=2)，一个Hash卷
  - 创建复制卷：
    - 在master1和agent1上创建`/data/repli`目录
    - 执行`gluster volume create replica_volume replica 2 transport tcp master1:/data/repli agent1:/data/repli force`创建名为replica_volume的复制卷
    - 执行`gluster volume start replica_volume`开启复制卷
    - 在master1上创建目录`/data/mnt`用于挂载复制卷
    - 执行`mount -t glusterfs master1:replica_volume /data/mnt`挂载复制卷
  
  - 创建Hash卷
    - 分别在四台机器上创建`/data/hash`目录
    - 执行`gluster volume create hash_volume master1:/data/hash agent1:/data/hash agent2:/data/hash agent3:/data/hash force `创建名为hash_volume的Hash卷
    - 执行`gluster volume start hash_volume`
    - 在master1上创建目录`/data/mnt_hash`用于挂载Hash卷
    - 执行`mount -t glusterfs master1:hash_volume /data/mnt_hash`挂载Hash卷


## Mesos+Zookeeper+Marathon集群搭建
### Mesos-master节点
这里使用master1节点作为mesos-master，配置如下：
- 关闭selinux
  ```bash
  sed -i '/SELINUX/s/enforcing/disabled/' /etc/selinux/config
  setenforce 0
  ```
- 关闭防火墙
  ```bash
  systemctl stop firewalld.service
  systemctl disable firewalld.service
  ```
- 添加mesos的yum源
  ```bash
  rpm -Uvh http://repos.mesosphere.io/el/7/noarch/RPMS/mesosphere-el-repo-7-1.noarch.rpm
  ```
- 安装依赖的JDK环境、Mesos1.5.0、Marathon、Zookeeper
  ```bash
  # 需要java8
  yum install -y java-1.8.0-openjdk-devel java-1.8.0-openjdk      
  yum -y install mesos marathon mesosphere-zookeeper
  ```
- 配置Zookeeper
我们将master1、agent1和agent2作为zookeeper的三个Znode，分别编号为1,2,3；zookeeper的配置必须在这三个Znode上都完成。
  - 将编号写到/var/lib/zookeeper/myid文件中去：
      ```bash
      master1: echo 1 > /var/lib/zookeeper/myid
      agent1: echo 2 > /var/lib/zookeeper/myid
      agent2: echo 3 > /var/lib/zookeeper/myid
      ```

  - 修改zookeeper文件：
    ```bash
    # 备份zoo.cfg文件
    cp /etc/zookeeper/conf/zoo.cfg /etc/zookeeper/conf/zoo.cfg.bak
    # 然后往zoo.cfg文件中加入以下配置信息：
    server.1=master1:2888:3888
    server.2=agent1:2888:3888
    server.3=agent2:2888:3888 
    ```
- 配置Mesos-master
  - 在/etc/mesos/zk文件中加入`zk://172.0.0.1:2181,172.0.0.2:2181,172.0.0.3:2181/mesos`
  - 设置文件/etc/mesos-master/quorum内容为一个大于（master节点数除以2）的整数。这里只有一个master，故quorum设置为1：`echo 1 >/etc/mesos-master/quorum`。
  - 创建目录：`mkdir -p /etc/marathon/conf`
  - 设置mesos-master的hostname：`echo 172.0.0.1 > /etc/mesos-master/hostname`
  - 设置marathon的hostname：`echo 172.0.0.1 > /etc/marathon/conf/hostname`
  - 在/etc/marathon/conf/master文件中加入`zk://172.0.0.1:2181,172.0.0.2:2181,172.0.0.3:2181/mesos`
  - 在/etc/marathon/conf/zk文件中加入`zk://172.0.0.1:2181,172.0.0.2:2181,172.0.0.3:2181/marathon`

- 启动mesos，marathon，zookeeper
  ```bash
  systemctl enable zookeeper && systemctl enable mesos-master && systemctl enable marathon
  systemctl start zookeeper && systemctl start mesos-master && systemctl start marathon
  ```
- 启动marathon的时候报错：未指定master。
  解决办法：在启动marathon的时候指定master和zk，即进入`/usr/lib/systemd/system/marathon.service`，修改
  ```bash
  ExecStart=/usr/share/marathon/bin/marathon \
  --master zk://172.0.0.1:2181,172.0.0.2:2181,172.0.0.3:2181/mesos \
  --zk zk://172.0.0.1:2181,172.0.0.2:2181,172.0.0.3:2181/marathon
  ```
  ```bash
  systemctl daemon-reload
  systemctl start marathon
  ```

### Mesos-slave节点
这里使用全部四个节点作为mesos-slave，配置如下：
- 关闭selinux
  ```bash
  sed -i '/SELINUX/s/enforcing/disabled/' /etc/selinux/config
  setenforce 0
  ```
- 关闭防火墙
  ```bash
  systemctl stop firewalld.service
  systemctl disable firewalld.service
  ```
- 添加mesos的yum源
  ```bash
  rpm -Uvh http://repos.mesosphere.io/el/7/noarch/RPMS/mesosphere-el-repo-7-1.noarch.rpm
  ```
- 安装Mesos1.5.0
  ```bash
  yum install -y mesos
  ```
- 将IP写入`/etc/mesos-slave/hostname`中：
  ```bash
  master1: echo 172.0.0.1 > /etc/mesos-slave/hostname
  agent1: echo 172.0.0.2 > /etc/mesos-slave/hostname
  agent2: echo 172.0.0.3 > /etc/mesos-slave/hostname
  agent3: echo 172.0.0.4 > /etc/mesos-slave/hostname
  ```
- 配置Mesos
  ```bash
  echo zk://172.0.0.1:2181,172.0.0.2:2181,172.0.0.3:2181/mesos /etc/mesos/zk
  ```
- 配置marathon调用mesos运行docker容器
  ```bash
  echo 'docker,mesos' > /etc/mesos-slave/containerizers
  ```
- 启动slave
  ```bash
  systemctl enable mesos-slave && systemctl start mesos-slave
  ```
  除了master1节点外，其余节点分别执行：`systemctl disable mesos-master`

### 遇到的问题：
  `Mesos-slave：Failed to perform recovery: Incompatible agent info detected.`通过`journalctl -xe`查看log，提示
  ```bash
  To restart this agent with a new agent id instead, do as follows:
  4月 03 19:13:30 iZbp194ugtbu9s8ct8lt3eZ mesos-slave[1542]: rm -f /var/mesos/meta/slaves/latest
  4月 03 19:13:30 iZbp194ugtbu9s8ct8lt3eZ mesos-slave[1542]: This ensures that the agent does not recover old live executors.
  ```
  执行`rm -f /var/mesos/meta/slaves/latest`，重启mesos-slave即可


## 安装并配置Etcd3.2.15
### 安装etcd
```bash
yum install etcd -y
```
### 配置etcd
- 修改etcd默认配置文件/etc/etcd/etcd.conf
  ```bash
  master1节点：
  #[Member]
  ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
  ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
  ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379,http://0.0.0.0:4001"
  ETCD_NAME="master1"

  #[Clustering]
  ETCD_INITIAL_ADVERTISE_PEER_URLS="http://master1:2380"、
  ETCD_ADVERTISE_CLIENT_URLS="http://master1:2379,http://master1:4001"
  ETCD_INITIAL_CLUSTER="master1=http://master1:2380,agent1=http://agent1:2380,agent2=http://agent2:2380,agent3=http://agent3:2380"
  ETCD_INITIAL_CLUSTER_TOKEN="mritd-etcd-cluster"
  ETCD_INITIAL_CLUSTER_STATE="new"

  agent1节点：
  #[Member]
  ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
  ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
  ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379,http://0.0.0.0:4001"
  ETCD_NAME="agent1"

  #[Clustering]
  ETCD_INITIAL_ADVERTISE_PEER_URLS="http://agent1:2380"
  ETCD_ADVERTISE_CLIENT_URLS="http://agent1:2379,http://agent1:4001"
  ETCD_INITIAL_CLUSTER="master1=http://master1:2380,agent1=http://agent1:2380,agent2=http://agent2:2380,agent3=http://agent3:2380"
  ETCD_INITIAL_CLUSTER_TOKEN="mritd-etcd-cluster"
  ETCD_INITIAL_CLUSTER_STATE="new"

  agent2节点：
  #[Member]
  ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
  ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
  ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379,http://0.0.0.0:4001"
  ETCD_NAME="agent2"

  #[Clustering]
  ETCD_INITIAL_ADVERTISE_PEER_URLS="http://agent2:2380"
  ETCD_ADVERTISE_CLIENT_URLS="http://agent2:2379,http://agent2:4001"
  ETCD_INITIAL_CLUSTER="master1=http://master1:2380,agent1=http://agent1:2380,agent2=http://agent2:2380,agent3=http://agent3:2380"
  ETCD_INITIAL_CLUSTER_TOKEN="mritd-etcd-cluster"
  ETCD_INITIAL_CLUSTER_STATE="new"

  agent3节点：
  #[Member]
  ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
  ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
  ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379,http://0.0.0.0:4001"
  ETCD_NAME="agent3"

  #[Clustering]
  ETCD_INITIAL_ADVERTISE_PEER_URLS="http://agent3:2380"
  ETCD_ADVERTISE_CLIENT_URLS="http://agent3:2379,http://agent3:4001"
  ETCD_INITIAL_CLUSTER="master1=http://master1:2380,agent1=http://agent1:2380,agent2=http://agent2:2380,agent3=http://agent3:2380"
  ETCD_INITIAL_CLUSTER_TOKEN="mritd-etcd-cluster"
  ETCD_INITIAL_CLUSTER_STATE="new"
  ```

- 修改/usr/lib/systemd/system/etcd.service:
  ```bash
  所有四个节点：
  [Unit]
  Description=Etcd Server
  After=network.target
  After=network-online.target
  Wants=network-online.target

  [Service]
  Type=notify
  WorkingDirectory=/var/lib/etcd/
  EnvironmentFile=-/etc/etcd/etcd.conf
  User=etcd
  # set GOMAXPROCS to number of processors
  ExecStart=/bin/bash -c "GOMAXPROCS=$(nproc) /usr/bin/etcd --name=\"${ETCD_NAME}\" \
  --data-dir=\"${ETCD_DATA_DIR}\" \
  --listen-client-urls=\"${ETCD_LISTEN_CLIENT_URLS}\" \
  --advertise-client-urls=\"${ETCD_ADVERTISE_CLIENT_URLS}\" \
  --initial-advertise-peer-urls=\"${ETCD_INITIAL_ADVERTISE_PEER_URLS}\" \
  --listen-peer-urls=\"${ETCD_LISTEN_PEER_URLS}\" \
  --initial-cluster-token=\"${ETCD_INITIAL_CLUSTER_TOKEN}\" \
  --initial-cluster=\"${ETCD_INITIAL_CLUSTER}\" \
  --initial-cluster-state=\"${ETCD_INITIAL_CLUSTER_STATE}\" \
  "
  Restart=on-failure
  LimitNOFILE=65536

  [Install]
  WantedBy=multi-user.target
  ```

- 开启etcd服务：
  ```bash
  systemctl daemon-reload
  systemctl start etcd
  ```

### 遇到的问题：
`error verifying flags, --advertise-client-urls is required when --listen-client-urls is set explicitly. See 'etcd --help'.`问题是--advertise-client-urls未指定。查看`/etc/etcd/etcd.cnf`发现`ETCD_ADVERTISE_CLIENT_URLS="http://master1:2379,http://master1:4001"`写成了`ETCD_ADVERTISE_CLIENT_URLS="http://master1:2379,master1:4001"`，应该加上`http://`

## 安装Calico
以下配置在每台机器上都需要完成。
- 下载calicoctl二进制文件
  ```bash
  wget -O /usr/local/bin/calicoctl https://github.com/projectcalico/calicoctl/releases/download/v1.6.3/calicoctl
  chmod +x /usr/local/bin/calicoctl
  ```
- 开启calico/node
  ```docker
  # master1节点
  ETCD_ENDPOINTS=http://master1:2379 calicoctl node run --node-image=quay.io/calico/node:v2.6.8

  # agent节点（使用自己搭建的registry仓库，下面将介绍如何搭建）
  # 将"insecure-registries":["master1:5000"]写入/etc/docker/daemon.json
  ETCD_ENDPOINTS=http://master1:2379 calicoctl node run --node-image=master1:5000/calico/node:v2.6.8
  ```
  
  检查calico/node是否运行：
  ```bash
  # docker ps
  CONTAINER ID  IMAGE                        COMMAND         CREATED        STATUS       PORTS     NAMES
  7cfe445e5d58  quay.io/calico/node:v2.6.8   "start_runit"   26 hours ago   Up 26 hours            calico-node

  # calicoctl node status
  Calico process is running.
  ```
  对于agent节点，从后面搭建的docker registry拉取镜像。
- 配置CNI网络：
  - 设置环境变量
    ```bash
    # set cni network
    # CNI configuration directory
    export NETWORK_CNI_PLUGINS_DIR=/var/lib/mesos/cni/plugins
    # CNI plugin directory
    export NETWORK_CNI_CONF_DIR=/var/lib/mesos/cni/config
    ```
  - 下载Calico CNI插件到$NETWORK_CNI_PLUGINS_DIR
    ```bash
    curl -L -o $NETWORK_CNI_PLUGINS_DIR/calico \
    https://github.com/projectcalico/cni-plugin/releases/download/v1.11.4/calico
    curl -L -o $NETWORK_CNI_PLUGINS_DIR/calico-ipam \
        https://github.com/projectcalico/cni-plugin/releases/download/v1.11.4/calico-ipam
    chmod +x $NETWORK_CNI_PLUGINS_DIR/calico
    chmod +x $NETWORK_CNI_PLUGINS_DIR/calico-ipam
    ```
  - 创建Calico CNI配置到$NETWORK_CNI_CONF_DIR
```bash
cat > $NETWORK_CNI_CONF_DIR/calico.conf <<EOF
{
  "name": "calico",
  "cniVersion": "0.1.0",
  "type": "calico",
  "ipam": {
      "type": "calico-ipam"
  },
  "etcd_endpoints": "http://master1:2379"
}
EOF
```
- Docker Registry的搭建(后面由marathon运行docker registry，该步骤可忽略)
  由于agent[1,2,3]不能出外网，故需要在master1节点上搭建docker registry。
  - 在master1节点上，执行`docker pull registry`拉取最新版docker registry镜像。
  - 创建/data/registry/config.yml文件，写入一下内容，在创建registry后允许删除镜像
    ```yml
    version: 0.1
    log:
      fields:
        service: registry
    storage:
      delete:
        enabled: true
      cache:
        blobdescriptor: inmemory
      filesystem:
        rootdirectory: /var/lib/registry
    http:
      addr: :5000
      headers:
        X-Content-Type-Options: [nosniff]
    health:
      storagedriver:
        enabled: true
        interval: 10s
        threshold: 3
    ```
  - 创建/opt/data/registry目录用于挂载registry容器内部存储的镜像
  - 运行registry
    加入insecure-registries到/etc/docker/daemon.json中：
    ```json
    {
        "graph": "/mnt/docker-data",
        "storage-driver": "overlay",
        "insecure-registries":[
            "master1:5000"
        ]
    }
    ```
    运行registry，将registry容器命名为docker_registry
    ```docker
    docker run -d -p 5000:5000 -v /opt/data/registry:/var/lib/registry  -v /data/registry/config.yml:/etc/docker/registry/config.yml --name docker_registry registry 
    ```
- Marathon运行Docker registry
  - 在/root/zhongchuang-cluster/marathon/registry.json文件中加入以下内容：
    ```json
    {
      "id": "registry",
      "cpus": 0.3,
      "mem": 100.0,
      "disk": 20000,
      "instances": 1,
      "constraints": [
        [
          "hostname",
          "CLUSTER",
          "172.0.0.1"
        ]
      ],
      "container": {
        "type": "DOCKER",
        "docker": {
          "image": "registry",
          "privileged": true
        },
        "volumes":[
          {
            "containerPath":"/var/lib/registry",
            "hostPath":"/opt/data/registry",
            "mode":"RW"
          },
          {
            "containerPath":"/etc/docker/registry/config.yml",
            "hostPath":"/data/registry/config.yml",
            "mode":"RO"
          }
        ],
        "portMappings": [
          { "containerPort": 5000, "hostPort": 5000 }
        ]
      }
    }
    ```
    /etc/docker/registry/config.yml为上一步创建的config.yml
  - 执行`curl -X POST http://master1:8080/v2/apps -d@registry.json -H "Conten-type:application/json"`即在marathon上启动docker registry。
  - 测试registry是否创建成功
    - `docker tag busybox master1:5000/busybox`
    - `docker push master1:5000/busybox`
    - 在agent1上执行：`docker pull master1:5000/busybox`成功。
  
- Marathon运行Marathon-lb：
  ```json
  {
    "id": "/system/marathon-lb",
    "cmd": null,
    "cpus": 1,
    "mem": 128,
    "disk": 0,
    "instances": 1,
    "constraints": [
      [
        "hostname",
        "CLUSTER",
        "172.0.0.1"
      ]
    ],
    "acceptedResourceRoles": [
      "*"
    ],
    "container": {
      "type": "DOCKER",
      "volumes": [],
      "docker": {
        "image": "docker.io/mesosphere/marathon-lb",
        "network": "HOST",
        "privileged": true,
        "parameters": [],
        "forcePullImage": false
      }
    },
    "portDefinitions": [
      {
        "port": 10000,
        "protocol": "tcp",
        "name": "default",
        "labels": {}
      }
    ],
    "args": [
      "sse",
      "-m",
      "http://master1:8080",
      "-m",
      "http://agent1:8080",
      "-m",
      "http://agent2:8080",
      "-m",
      "http://agent3:8080",
      "--group",
      "external"
    ],
    "healthChecks": []
  }
  ```
  - 遇到的问题：marathon前端一直显示waiting。解决方案是：
    - mesos-slave上资源不够，一般是内存不够。可上mesos-master:5050上查看
    - 宿主机上没有镜像，一直在拉或拉不到。上宿主机上查看: docker images | grep xxx，确保marathon上配置的镜像名和版本在宿主机上存在
    - marathon上属性Constraints中包含的主机名和/etc/mesos-slave/hostname的内容一样。一开始配置constraints的时候将172.0.0.1写成了master1，而/etc/mesos-slave/hostname写的是172.0.0.1
    - mesos-master数量有问题，正常情况应该是3个，至少需要是奇数。如果有多余启动的mesos-master，关掉服务并禁止启动

## 安装DC
- 遇到的问题：阿里云上haproxy/nginx反代到自己机子上的一个端口时，如果upstream用的公网ip，会被莫名的拦截，即使开了安全组。
  - 使用内网IP做反向代理的upstream就没问题。
