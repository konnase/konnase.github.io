---
title: Kubernetes搭建
p: resource_scheduling/kubernetes-搭建
date: 2018-07-09 14:19:40
tags: [kubernetes]
categories: Resource Management
typora-root-url: ../../../source
---

# 使用kubeadm安装kubernetes
## 预备条件
- /etc/hosts文件
- 关闭firewall
- 关闭selinux
- 关闭swap
- iptables参数
```bash
$ cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

$ sysctl --system
```
<!--more-->
## 按照官网安装kubernetes
[create-cluster-kubeadm](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/)

确保docker和kubelet使用相同的cgroup driver(cgroupfs or systemd)

有些镜像需要外网才能pull下来，可以使用shadowsocks开一个http代理，服务器指定http代理pull镜像（注意接网线，要不然找不到地址。。）

执行
```bash
$ kubectl init --kubernetes-version=v.1.11.0 --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=[master-ip]
```
关闭http_proxy，不然最后init不成功，`--kubernetes-version`需要加上，不然需要连接外网获取version。总结就是要连外网的时候挂代理，不用的时候（连接集群中的某些机器时）把代理去掉。

`--pod-network-cidr=10.244.0.0/16`的意思是设置cidr（无类别域间路由），使10.244.0.0/16网段的路由都通过10.244.0.0网络号转发（就是所谓的网络地址斜杠标记法），在node加入k8s集群之后，pod network(eg. flannel)会自动为其他node分配一个10.244.0.0/16网段的一个子网，如10.244.1.0/24

安装dashboard后，要通过其他机器浏览器访问ui，需要开启kube-proxy，执行
```bash
$ kubectl proxy --address 0.0.0.0 --port=8181 --accept-hosts='^*$'
```
保证接受任何ip、任何host发送的请求

## 坑
### 镜像需要从外网获取
执行`kubectl get pods --all-namespaces`观察到有很多pods的状态不是running，其实是一些镜像没能pull下来
![image](/img/pod_error.png)
将这些镜像pull下来传到registry，然后每个node上从registry pull镜像，再tag称官方的镜像名。

kubernetes-dashboard也一样要pull镜像下来，开启kube-proxy后，浏览器访问 [n167:8181/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy](http://n167:8181/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/) 或者[n167:6443/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/](n167:6443/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/)即可进入登录界面。（提示services \"https:kubernetes-dashboard:\" is forbidden: User \"system:anonymous\" cannot get services/proxy in the namespace \"kube-system\"时表示用户未认证，需要生成证书，使用openssl生成p12证书，然后在浏览器导入该证书，才能从个人电脑上访问服务器的dashboard）

### pod network
不要单独用calico做proxy和networking，否则k8s集群内部pods之间无法通信，最大的影响是heapster不能集成，看不到配置字典等。推荐使用canal pod network，它使用calico做proxy，flannel做networking。

**targetPort表示容器内部的端口，port表示Service的虚拟端口，不要搞反了**；targetPort只在service起作用，表示用来访问该service对应的容器的内部containerPort；外部系统只要用任意一个Node的IP地址+nodePort即可访问到相应的服务（即使该服务只运行在一个node上，也可以通过任意一个node访问到该服务）

### cluster reset
集群重置过后，再从浏览器登录的时候，由于之前的证书已经失效，需要重新导入证书，将之前的证书删除

### Service
cluster IP无法被ping通，因为没有“实体对象”响应单独的clusterIP，不具备TCP/IP通信的基础。当通过域名（如nginx.default.svc）访问应用时，实际上是通过service访问的，k8s内部会将该域名解析为service的clusterIP，所以在集群中是不能ping通某个域名，但是可以走http协议（如curl命令）拿到该域名对应端口的内容，注意端口应该使用Service的虚拟端口port，而不是容器内部的端口targetPort（如nginx.default.svc:8080）

kubernetes 集群中service可以映射的物理机端口范围默认为30000~32767

Replica Set就是Replication Controller，只是比后者多了一个支持集合的Label Selector，RS一般不单独使用，主要是被Deployment这个更高层的资源对象使用，RS负责Deployment的自动扩容

### 三种方式访问nginx服务器
- podIP:port  只能在本机上访问
- clusterIP:targetPort  只能在集群内访问
- nodeIP:nodePort  可以在外网访问（只要nodeIP可以出外网）

### kubelet
kubelet需要一个read-only-port，监听其他pod的请求，使用安装完kubelet之后默认为启用该port。在`/etc/systemd/system/kubelet.service.d/10.kubeadm.conf`中加入`Environment="KUBELET_READ_ONLY_PORT=--read-only-port=10255"`，并在Execstart后接上该环境变量`$KUBELET_READ_ONLY_PORT`，完整的文件如下：
```conf
# Note: This dropin only works with kubeadm and kubelet v1.11+
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
# This is a file that "kubeadm init" and "kubeadm join" generates at runtime, populating the KUBELET_KUBEADM_ARGS variable dynamically
EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env
# This is a file that the user can use for overrides of the kubelet args as a last resort. Preferably, the user should use
# the .NodeRegistration.KubeletExtraArgs object in the configuration files instead. KUBELET_EXTRA_ARGS should be sourced from this file.
EnvironmentFile=-/etc/sysconfig/kubelet
Environment="KUBELET_READ_ONLY_PORT=--read-only-port=10255"
ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS $KUBELET_READ_ONLY_PORT
```

### 更换网卡
需要修改flannel网络
- 到～/kubernetes/flannel下面修改kube-flannel.yml，因为机器是amd64的，所以修改DaemonSet kube-flannel-ds-amd64的container kube-flannel的args的--iface=p3p2，改成某一张网卡设备
- 重启每台机器上的kube-flannel
- 如果遇到同一台机器上两个service之间不能通信的问题，可能是iptables被跳过了，应该将下面两项置为1
```bash
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```
- 改变每台机器上的internal ip：在/etc/hosts里面加入
```
10.0.1.167  n167.njuics.cn
10.0.1.168  n168.njuics.cn
10.0.1.169  n169.njuics.cn
10.0.1.170  n170.njuics.cn
```