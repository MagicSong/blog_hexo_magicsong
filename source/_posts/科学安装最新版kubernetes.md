---
title: 离线安装最新版kubernetes（1.10）
date: 2018-04-08 09:12:43
tags: 
    - kubernetes
    - k8s
    - docker
    - 安装
categories: docker
---

> 随着docker的流行，越来越多企业采用了容器技术作为IT部门的基础架构，越来越多的运用在了生产环境中。生产环境中各个应用都是以微服务的形式分布在各个主机中，k8s就是这样用于在分布式系统中管理容器的一种编排工具，提供基本
的部署，维护以及运用伸缩。
 <!-- more -->
# 1. 简介
**Kubernetes **是 Google 团队发起的开源项目，它的目标是管理跨多个主机的容器，主要实现语言为 Go 语言。Kubernetes 构建于 Google 数十年经验，一大半来源于 Google 生产环境规模的经验。结合了社区最佳的想法和实践。在分布式系统中，部署，调度，伸缩一直是最为重要的也最为基础的功能。Kubernets 就是希望解决这一序列问题的。Kubernetes 有如下这些特点：
+ 易学：轻量级，简单，容易理解
+ 便携：支持公有云，私有云，混合云，以及多种云平台
+ 可拓展：模块化，可插拔，支持钩子，可任意组合
+ 自修复：自动重调度，自动重启，自动复制

也是由于这些特点，k8s已经成为大多数企业的容器编排工具，而docker官方的**docker swarm** 则由于后知后觉的缘故而使用较少（docker swarm是最新版本的docker自带的，无需安装，使用简单，初学容器编排时可以尝试一下）。
> 学习k8s有一定的难度，本文着重于介绍如何搭建一个可用的k8s集群，后续会写一些文章重点学习k8s的一些key concept。

# 2. 安装Kubernetes
k8s官方文档中介绍了很多种部署k8s集群的方式，有单机实验性质的，本地多机部署方式，也有云端部署的方式，本文介绍的是第二种**本地多机部署**——即利用本地多台机器部署一个k8s集群，其他方式可以参看[官方文档](https://kubernetes.io/docs/home/)。需要注意的是，官方文档提供的部署方式由于一些不可描述原因在我国实施起来有一定难度，所以本文的安装需要**科学上网**。并且，本次实验的三台主机都是在局域网中，没有联机权限，所以并不要求服务器能够科学上网。
在我安装的时候，最新版的k8s的版本是`1.10`，和其他版本的安装过程是相似的。

##  Prerequisites 检查

在参照本文部署最新版k8s前，请一定要注意k8s安装的前置条件，现在罗列如下，请依依对照，一条不可漏下。

1. 有一台能够科学上网的机器，下载相关的软件和镜像
2. 下列操作系统之一
    + Ubuntu 16.04+
    + Debian 9
    + CentOS 7
    + RHEL 7
    + Fedora 25/26 (best-effort)
    + HypriotOS v1.0.1+
    + Container Linux (tested with 1576.4.0)
3. 每台主机至少2G内存，双核
4. 集群中所有主机互相访问没有限制（无论是公网还是私有网卡）**UPDATE**，这里我踩了一个超级大坑，我司申请的两台机器互相访问无限制往往是TCP端口无限制，但是k8s需要用到UDP。所以一定要确保节点之间UDP访问也无限制！
5. 主机名，MAC地址，产品UUID必须不同（一般都是不同的，除非是用的是复制的虚拟机）
6. 如果使用了防火墙，必须打开一些指定端口（下文会讲到，如果嫌麻烦，直接关了防火墙，本次实验没有关闭防火墙）
7. 禁用SWAP内存。可以使用命令`swapoff -a`，其他方式也可以。

下面列出安装k8s会使用到端口，如果使用防火墙的话，请将这些端口打开（我在实验时遇到了一个坑就是端口问题，我在防火墙配置了三个节点互相访问没有限制，但是这还不是不够的，一些容器是通过虚拟网卡访问接口的，所以还是需要打开一些额外的端口。这里事先记录一下，如果在安装过程中打开了防火墙，并且也配置了三个节点ip互相访问没有限制，那么还需要再打开的默认端口——在不修改默认端口的前提下——6443，10250-10252，请确保这些端口对公可访问，**在初始化的时候会提示你打开这些端口**）。
**master节点**

| 协议 | 方向    | 端口      | 目的                    |
| ---- | ------- | --------- | ----------------------- |
| TCP  | Inbound | 6443      | k8s api 服务器          |
| TCP  | Inbound | 2379-2380 | etcd 客户端API          |
| TCP  | Inbound | 10250     | Kubelet API             |
| TCP  | Inbound | 10251     | kube-scheduler          |
| TCP  | Inbound | 10252     | kube-controller-manager |
| TCP  | Inbound | 10255     | Read-only Kubelet API   |

**worker节点**

| 协议 | 方向    | 端口        | 目的                  |
| ---- | ------- | ----------- | --------------- |
| TCP  | Inbound | 10250       | Kubelet API           |
| TCP  | Inbound | 10255       | Read-only Kubelet API |
| TCP  | Inbound | 30000-32767 | 服务端口              |

*如果需要修改一些默认端口，那么需要确保修改后的端口也打开。*

## 安装docker

docker 的安装可以参考此文：[docker部署指南](http://itdocker.rd.tp-link.net:9002/2018/03/14/Docker%E9%83%A8%E7%BD%B2%E6%8C%87%E5%8D%97/)
如果是运行在公网中的服务器，请查看官方文档。

## 安装kudeadm
本次实验使用了三台centos7.4的主机，安装的docker使用了最新的docker-ce。

kubeadm 是google公司用来简化k8s集群搭建的一个工具，安装完kubeadm之后，只需要一行命令就可以初始化一个集群的master节点，非常方便。安装kubeadm的时候，还需要安装kubelet和kubectl，这两个工具是用来管理集群的工具，kubeadm本身不会安装这两个工具。需要注意kubeadm和这两工具的版本需要一致，此次的版本都是1.10。由于服务器不能上网，此次我们需要手动安装rpm包。在一台能科学上网的机器上输入以下网址（这个网址其实就是google提供的yum源中软件的安装位置）：https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64/repodata/primary.xml
这个xml文件中包括了所有需要的rpm包和其依赖项，本次安装需要用到下面4个包：

* kubeadm
* kubectl 
* kubelet
* kubernetes-cni 

请在上面的xml找到上面软件的最新版（大部分是1.10，如果不是，请确保是在上述xml文件中是最新的），复制其对应的地址进行下载，下载完成之后将rpm包传输到所有服务器节点即可。本次下载的截图如下：
{% asset_img 2018-04-08_114922.jpg 所需的安装包 %}
下载完成之后，在所有节点安装这些程序：
``` bash
setenforce 0
yum install -y *.rpm
systemctl enable kubelet && systemctl start kubelet
```
## 后续配置
> 安装完成之后需要进行一系列和k8s配置相关的后续操作。有一些操作需要在所有节点进行，而有一些只需要在master 节点进行。

需要注意`setenforce 0`，目前k8s的大部分网络插件都无法在selinux下运行，所以需要将selinux关闭。如果在安装网络插件的过程中遇到问题。可以检查一下是否是这个问题（当初就是踩了这个坑）。另外如果和我一样操作系统也是centos7，那么还需要进行下面的配置：
```bash
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```

请注意。上述所有操作在**所有节点**都要进行。**网络插件的安装我遇到了不少问题，这部分大家可以参考我的，一步一步谨慎操作。**
除了网络配置，还需要检查主节点k8s cgroup driver是否和docker 的cgroup driver相同，如果不相同需要相同。这一步只需要在主节点完成。
```bash
#检查下面两个命令的输出
docker info | grep -i cgroup
cat /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

#如果不相同，需要将k8s的配置改为和docker一致，执行下面的命令
sed -i "s/cgroup-driver=systemd/cgroup-driver=cgroupfs/g" /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
```
上述后续配置都完成之后，重启kubelet
```bash
systemctl daemon-reload
systemctl restart kubelet
```
如果没出现什么问题，那么kubeadm的安装已经完成，下面就需要用kubeadm创建一个集群。

## 用kubeadm创建一个集群
**1. 阅读相关网络插件说明**

kubeadm 极大地简化了创建一个k8s集群的步骤，但它本身没有能力提供集群的网络解决方案。我们需要选择一个**合适**的网络解决方案。在熟悉k8s之前，我们无法判断一个网络解决方案到底合不合适，所以本次实验选择了一个比较容易上手的网络插件——flannel。不同的网络插件下，k8s初始化的步骤不同，请一定要参考相关的文档进行设置。

**2. 手动拉取一些基础镜像**

kubeadm在初始化的过程中会启动一系列docker镜像（而且是墙外的镜像），由于我们的服务器无法联网，我们需要在一个能够科学上网的机器上安装docker，并拉取相关的镜像。需要拉取的镜像列表如下，我用的tag是latest，目前没有遇到什么问题：

* k8s.gcr.io/kube-proxy-amd64
* k8s.gcr.io/kube-controller-manager-amd64
* k8s.gcr.io/kube-apiserver-amd64
* k8s.gcr.io/kube-scheduler-amd64
* k8s.gcr.io/etcd-amd64
* k8s.gcr.io/kubernetes-dashboard-amd64
* k8s.gcr.io/k8s-dns-dnsmasq-nanny-amd64
* k8s.gcr.io/k8s-dns-sidecar-amd64
* k8s.gcr.io/k8s-dns-kube-dns-amd64
* k8s.gcr.io/pause-amd64

除了上述基础镜像，还需要拉取网络插件镜像，取决于所采用的网络方案，例如本次下载的镜像是：quay.io/coreos/flannel。如何知道k8s需要哪些镜像呢？其实如果不事先拉取镜像，直接运行`kubeadm init`，会在`/etc/kubernetes/manifests/`目录下生成一系列yml文件，这些yml文件告知了k8s在启动过程中需要哪些镜像（如果一个一个去看了的话，你会发现上述有一个镜像并不是初始化集群必须的。哈哈，不不卖关子了，就是那个dashboard，这只是一个管理k8s的web ui）。
如果你没有拉取镜像，而直接运行了`kubeadm init`，肯定会报错，报错之后如果想要重新安装，请一定要执行`kubeadm reset`

拉取完上述镜像之后，将上述镜像传输到主节点中。还有一部分镜像需要传输到从节点中，列表如下：
    * k8s.gcr.io/kube-proxy-amd64
    * k8s.gcr.io/kubernetes-dashboard-amd64
    * k8s.gcr.io/pause-amd64
    * quay.io/coreos/flannel

在局域网中，镜像共享需要私有镜像。传输镜像的步骤为，tag镜像，然后push到私有仓库，或者也可以采用下面的方式直接传输，避免需要不断重复tag。
```bash
docker save k8s.gcr.io/pause-amd64:3.1 | bzip2 | pv | ssh root@nodeip 'cat | docker load'
#pv可以将进度可视化，需要安装一下，没有pv也一样OK
```
**3. 初始化集群**

首先在主节点运行下面的命令初始化为一个集群：
```bash
kubeadm init --kubernetes-version=v1.10.0 --pod-network-cidr=10.244.0.0/16
```
`kubeadm init` 用于初始化一个集群，后面第一参数是显示告知集群的版本，如果不设置kubeadm会去网络上下载相关的版本信息，由于我们的服务器无法联网，这一步就会出错。所以一定要添加这个命令。第二个参数是flannel相关的参数设置，其中的地址是固定的10.244.0.0/16，这是在flannel配置文件中写死的。
上述命令的输出很重要，大部分应该下面这样的：
```
[init] Using Kubernetes version: v1.8.0
[init] Using Authorization modes: [Node RBAC]
[preflight] Running pre-flight checks
[kubeadm] WARNING: starting in 1.8, tokens expire after 24 hours by default (if you require a non-expiring token use --token-ttl 0)
[certificates] Generated ca certificate and key.
[certificates] Generated apiserver certificate and key.
[certificates] apiserver serving cert is signed for DNS names [kubeadm-master kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 10.138.0.4]
[certificates] Generated apiserver-kubelet-client certificate and key.
[certificates] Generated sa key and public key.
[certificates] Generated front-proxy-ca certificate and key.
[certificates] Generated front-proxy-client certificate and key.
[certificates] Valid certificates and keys now exist in "/etc/kubernetes/pki"
[kubeconfig] Wrote KubeConfig file to disk: "admin.conf"
[kubeconfig] Wrote KubeConfig file to disk: "kubelet.conf"
[kubeconfig] Wrote KubeConfig file to disk: "controller-manager.conf"
[kubeconfig] Wrote KubeConfig file to disk: "scheduler.conf"
[controlplane] Wrote Static Pod manifest for component kube-apiserver to "/etc/kubernetes/manifests/kube-apiserver.yaml"
[controlplane] Wrote Static Pod manifest for component kube-controller-manager to "/etc/kubernetes/manifests/kube-controller-manager.yaml"
[controlplane] Wrote Static Pod manifest for component kube-scheduler to "/etc/kubernetes/manifests/kube-scheduler.yaml"
[etcd] Wrote Static Pod manifest for a local etcd instance to "/etc/kubernetes/manifests/etcd.yaml"
[init] Waiting for the kubelet to boot up the control plane as Static Pods from directory "/etc/kubernetes/manifests"
[init] This often takes around a minute; or longer if the control plane images have to be pulled.
[apiclient] All control plane components are healthy after 39.511972 seconds
[uploadconfig] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[markmaster] Will mark node master as master by adding a label and a taint
[markmaster] Master master tainted and labelled with key/value: node-role.kubernetes.io/master=""
[bootstraptoken] Using token: <token>
[bootstraptoken] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstraptoken] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstraptoken] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: kube-dns
[addons] Applied essential addon: kube-proxy

Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run (as a regular user):

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  http://kubernetes.io/docs/admin/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join --token <token> <master-ip>:<master-port> --discovery-token-ca-cert-hash sha256:<hash>
```
上述输出告知了后续的操作，其中最重要的就是最后的token那段，那是节点加入整个集群的关键语句，需要保存以备后续节点加入使用，如果忘记了话可以用`kubeadm token`命令进行查询。上述命令成功的话不代表集群创建成功了，后续还需要安装网络插件。

（可选）如果需要以非root的身份使用k8s，那么需要切换到其用户下，执行下面的命令（该用户需要有sudo权限）：
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
如果是root用户，只需要在`.bash_profile`中添加下面一句即可。
```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
```
**4. 安装网络插件**

_网络插件_ 这一块是我踩坑的密集区，大部分都是和上面的讲到的步骤有关。上述讲到的步骤漏了一步都会导致网络插件安装失败。网络插件是必须的，集群中的应用需要通过网络插件进行沟通，**但是k8s并不自带网络插件**。本次选择了flannel作为我们的网络插件，由于不可联网，首先我们需要通过可以联网的将相关的yml文件下载到本地然后传输到服务器。<https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml>
然后执行`kubectl -f kube-flannel.yml` 开始部署网络插件。部署需要几秒钟，可以执行命令`kubectl get pods --all-namespaces`查看当前的部署状态，如果一段时间过去了所有的POD并不都是running状态，那么说明安装是失败的，那么需要一一检查上面说到步骤，检查kubeadm init的输出中有没有相关有用的信息。极有可能和防火墙以及selinux有关。

**5. 将主节点做为工作节点**

k8s不推荐将master节点做为工作节点，当然如果在资源有限的情况下，可以将主节点变为工作节点。执行下面的语句之后主节点也可以跑容器。
```bash
kubectl taint nodes --all node-role.kubernetes.io/master-
```

**6. 加入工作节点**

在工作节点上输入刚才保存的kubeadm join那一串就可以将一个节点加入到集群中，加入的语句是这样的：
```bash
kubeadm join --token <token> <master-ip>:<master-port> --discovery-token-ca-cert-hash sha256:<hash>
```
在**主节点**上执行`kubectl get nodes`可以看到当前的节点情况，如果很长一段时间都不是ready状态，说明加入有问题。我遇到过这个问题，经过一系列排查，发现是子节点上没有相关的镜像。按照上面提到的步骤，将一些镜像传输到节点上就可以了。如果遇到问题，执行`journalctl -r -u kubelet` 来找出问题所在吧（主节点和子节点的问题描述会不一致）！最终的输出如下：
```bash
kubectl get nodes
NAME                      STATUS    ROLES     AGE       VERSION
node1   Ready     <none>    3d        v1.10.0
node2   Ready     <none>    3d        v1.10.0
master   Ready     master    3d        v1.10.0
```
安装过程如果有错误，要先执行`kubeadm reset`，不然无法重新安装。

至此，一个集群的安装就已经完成了，后续将安装dashboard。dashboard虽然只是一个web ui，但是它有管理的权限，所以安装dashboard还需要额外的安全配置，内容详见下一篇文章。