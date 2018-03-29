---
title: 部署DC环境
date: 2018-03-29 15:12:49
tags: [docker, mesos, marathon, zookeeper, glusterfs, calico, docker registry]
---

# 基础环境搭建
## 安装docker
1. 安装必要的packages
```
sudo yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2
```
2. 配置阿里镜像源
```
sudo yum-config-manager \
    --add-repo \
    http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```
3. 安装docker
```
yum install docker-ce
```
4. 开启docker服务
```
systemctl start docker
```
### 遇到的问题：
docker daemon不能启动，即执行`systemctl start docker`时提示`Job for docker.service failed because the control process exited with error code. See "systemctl status docker.service" and "journalctl -xe" for details. `，执行`systemctl status docker.service `，输出` Active: failed (Result: start-limit)  Process: 7880 ExecStart=/usr/bin/dockerd (code=exited, status=1/FAILURE) Main PID: 7880 (code=exited, status=1/FAILURE)`。
解决方法：
```
rm -rf /var/lib/docker/
# 添加如下内容
# vim /etc/docker/daemon.json
{
    "graph": "/mnt/docker-data",
    "storage-driver": "overlay"
}
```