---
title: Docker部署指南
date: 2018-03-14 14:59:05
author: MagicSong
categories: docker
tags: 
    - 安装
    - 部署
    - docker
---

# IT Docker部署简单指南

## 一、环境准备

1. 我司普遍采用的是centos6，但是Centos7加强了很多虚拟化功能，所以建议在Centos7上部署。CentOS7和6的操作微微有一点差异，会有记号标注。
2. 关于联网，如果服务器能联网，那么下面的很多步骤可以省略，此次以我司无法连接外网的情况下进行配置。一样很简单。
<!-- more -->
***

## 二、安装Docker

Docker的安装可以有很多方式，最方便的还是直接利用yum安装。此次让网络管理课添加了Docker-CE的镜像了，可以直接安装，步骤如下：

1. 配置公司yum仓库和Docker仓库，一共有两个文件。前面一个内网上有，后面一个配置如下：

```bash
wget http://rdsource.tp-link.net/docker-ce/linux/centos/docker-ce.repo
###检查下载的文件中的网址是否正确，如果有里面的https://download.docker.com 则替换为http://rdsource.tp-link.net/docker-ce
mv docker-ce.repo /etc/yum.repos.d/
yum makecache
#用下面的指令查看是否有
yum list|grep docker-ce
```

由于centos7自带Docker，我们要首先卸载掉：

```bash
yum remove docker \
docker-common \
docker-selinux \
docker-engine
```

安装一些依赖包（可选，大部分时候都是自带）

```bash
yum install -y yum-utils \
device-mapper-persistent-data \
lvm2
```

安装docker

```bash
yum install docker-ce
```

配置Docker开机启动（*注：这里和6的操作不一样*）

```bash
systemctl enable docker
```

配置Docker仓库。我们的服务器连不上网，所以需要连接上公司私有的仓库。`vi /etc/docker/daemon.json`这个路径如果不存在的话新建就好，另外启动docker也会创建这个路径。由于配置仓库还需要重启，这里就不启动docker而手动创建了。在打开的文件中输入下面的内容。

```json
{
  "insecure-registries": [
    "rdsource.tp-link.net:8088"
  ],
  "disable-legacy-registry": true
}
```

启动docker

```bash
systemctl start docker
#配置用户名和密码
docker login -u admin -p admin123 rdsource.tp-link.net:8088
```

测试一下，拉取Hello World，然后运行

```bash
docker pull hello-world
docker run hello-world
```

如果出现下面的信息，代表Docker已经安装成功。

![image](http://itdbaas1.rd.tp-link.net:2000/api/getFile?filename=file_1519443696129.jpg)

**注意**:上面介绍的语法是默认，在我司无法连接外网的情况下，需要指定registry，我司的地址是rdsource.tp-link.net:8088，所以拉取hello-world的完整语句如下：

```bash
docker pull rdsource.tp-link.net:8088/hello-world
docker run rdsource.tp-link.net:8088/hello-world
```

***

## 三、配置管理镜像

上述配置都成功之后就需要安装docker的管理工具，由于本次的Docker主要用于测试环境，所以选择了轻量级的Portainer工具来管理Docker。***在这里要提一个非常有趣的东西***，因为Portainer官方提供的工具也是一个容器（当然也可以非容器），而由于容器访问不了外部环境，那么它是如何管理Docker的呢？docker社区提出了一个非常聪明的解决方案，将docker的socket挂载在容器内部一模一样的虚拟环境中。会有疑问，socket不是通信吗？怎么挂载？这就归功于linux的设计者提出的思想，**Everything is file**，上述的通信实际上可以通过/var/run/docker.sock访问，所以只需要将这个文件挂载在虚拟系统中，那么容器中也能管理本地docker啦。

### 1. 创建网络

在此之前还需要配置相关的网络，因为我们的portainer需要去访问另外一个容器的端口，所以需要将他们放入同一个网络。用下面的语句创建一个网络：

```bash
docker network create -d bridge IT_DOCKER_NET
```

创建完成之后，以后的容器在启动时只需要加入--network参数即可。

在配置管理工具之前先创建相关目录：

```bash
mkdir -p /opt/docker_data/portainer
```


### 2. 部署portainer容器

然后运行下面的命令下载并运行Portainer：

`docker run -d -p 9000:9000 --restart always -v /var/run/docker.sock:/var/run/docker.sock -v /opt/docker_data/portainer:/data --network IT_DOCKER_NET rdsource.tp-link.net:8088/portainer/portainer -t "http://localhost:9001/templates.json"`

上面的命令中需要解释的就是一个-t，这是portainer提供的模板功能，由于我们无法上网，所以需要搭建本地的模板网站（用的是nginx代理的），在启动的时候需要指向这个本地的网站。后续我们会讲如何搭建本地模板网站，在docker的帮助下非常简单。

在浏览器中输入地址+端口号9000进入管理页面，如下图：

![image](http://itdbaas1.rd.tp-link.net:2000/api/getFile?filename=file_1520240556949.jpg)

初始登录时设置管理员用户名和密码。下一步点击管理本地Docker，如下图：

![image](http://itdbaas1.rd.tp-link.net:2000/api/getFile?filename=file_1520240630812.jpg)

然后就能进入管理界面。

### 3.重要更新，利用docker-compose

上述部署中，portainer和templates是分开部署的，这在实际生产过程中用的比较少，通常更可靠的做法是用一些编排工具编制这些有联系的服务。官方推荐的工具时docker-compose，docker-compose能够将一些容器做成服务，然后组成一个完整的项目，具体的使用可以参考官方文档，这里简单介绍一下在ITDOCKER上的部署。

首先需要去[github](https://github.com/docker/compose/releases)上下载对应的二进制文件，已经编译好，可以直接放入相关bin目录中使用。下载完成之后记得需要`chmod +x`。运行`docker-compose --version`查看是否正常。

docker-compose也和dockerfile一样基于一个叫docker-compose.yml的文件，在工作目录下vim docker-compose.yml，输入以下内容：

```
version: "3"
services:
  portainer-web:
    image: "rdsource.tp-link.net:8088/portainer/portainer"
    command: --templates http://portainer-templates/templates.json
    restart: always
    ports: 
      - "9000:9000"
    volumes:
      - "/opt/docker_data/portainer:/data"
      - "/var/run/docker.sock:/var/run/docker.sock"
  portainer-templates:
    image: "portainer-templates"
    volumes:
      - "/opt/docker_data/portainer_template/templates.json:/usr/share/nginx/html/templates.json"
    expose: 
      - "80"
```

注意需要缩进，从文件中可以看出，portainer-web是我们的可视化管理工具，而portainer-templates是我们的模板仓库。

模板仓库暴露了80端口，但是只能由compose内部进行访问，外部是无法通过宿主机访问的。外部唯一能访问的就是9000端口。更方便的是前端管理界面可以直接通过服务名访问templates，compose帮我们做了相关的DNS设置。

输入`docker-compose up -d` 就可以启动整个应用了。进入管理界面，点击app templates如果看到下面的界面就表示部署成功了。

![image](http://itdbaas1.rd.tp-link.net:2000/api/getFile?filename=file_1520308152716.jpg)

之后就可以进入相关页面进行设置和管理。
