---
title: 学习在集群中工作—NFS文件共享
date: 2018-04-13 16:32:44
tags: 
    - linux
    - NFS
catagories:
    - 集群技术
---

> 刚进公司的时候第一次接触的服务器就是一个集群，一个RAC集群，为了集群正常工作，有一些文件需要在多个节点上一致，公司一些OA系统往往也需要所有节点的文件一致。以前采用的lsyncd+rsync的同步的方式进行工作，但是很容易出错，主要原因就是双相同步容易冲突，在高并发的时候就会出现异常。在学习docker集群的过程中也遇到了这些问题。由于一些服务需要在配置高可用（同时也是负载均衡），这些服务需要访问同一个文件目录。NFS能够解决这种文件共享问题。

# 1. 简介
NFS 是Network File System的缩写，即网络文件系统。功能是通过网络让不同的机器、不同的操作系统能够彼此分享文件，让应用程序在客户端通过网络访问位于服务器磁盘中的数据，是在类Unix系统间实现磁盘文件共享的一种方法。NFS使用RPC协议进行通信，也就是说NFS系统只是一组RPC程序。RPC是远程过程调用 (Remote Procedure Call) 的英文缩写，它是能使客户端执行其他系统中程序的一种机制。NFS可以看作是一个RPC Server，主要功能是管理需要分享的目录和文件。它不负责通信和信息传输，而是把这部分工作交给RPC协议来完成。即NFS在文件传送或信息传送过程中依赖于RPC协议。所以只要用到NFS的地方都要启动RPC服务，不论是NFS SERVER或者NFS CLIENT。这样SERVER和CLIENT才能通过RPC来实现PROGRAM PORT的对应。可以这么理解RPC和NFS的关系：NFS是一个文件系统，而RPC是负责负责信息的传输。
<!-- more -->
NFS 的基本原则是“容许不同的客户端及服务端通过一组RPC分享相同的文件系统”，它是独立于操作系统，容许不同硬件及操作系统的系统共同进行文件的分享。

# 2.搭建NFS文件服务器
## 安装软件
搭建NFS非常简单，本文的实验环境是centos 7.4，搭建了NFS文件服务器之后，只要有支持NFS的软件，任何系统都可以挂载NFS服务器挂载的目录。本次安装关闭了防火墙，如果开启了防火墙的话，需要打开对应端口。
执行下面的命令安装所需的软件：
```bash
yum install -y nfs-utils
systemctl enable rpcbind && systemctl start rpcbind
systemctl enable nfs && systemctl start nfs
```
上述命令会把NFS的依赖rpcbind也装上。安装之后直接启动NFS就可以了。

## 配置NFS
NFS的配置文件为`/etc/exports`，如果不存在这个文件，自己建一个即可。下面贴上我的配置：
```bash
/opt/docker_files 172.31.73.76(rw,no_root_squash,async) 192.168.22.0/24(rw,no_root_squash)
/opt/docker_data  192.168.22.0/24(rw,no_root_squash)
/opt/softwares 192.168.22.0/24(rw,no_root_squash)
```
配置文件非常简单，一行一个目录，后面跟上可以访问的地址即可，多个ip地址之间通过空格分开，ip地址可以是单独的ip（`172.31.73.76`是我的ip），也可以是一个域（`192.168.22.0/24`这个域包含了集群中的所有节点）。或者也可以用域名通配符，比如`*.rd.tp-link.net`。地址后面括号表示了访问权限以及如何处理访问的用户。rw表示可读可写，no_root_squash表示将所有使用这个目录的人当做root。使用这个目录的人就是当前挂载这个目录的主机正在使用这个文件夹的用户。上述配置在公网中是非常不安全的，它允许客户端root访问root文件夹，但是客户端如果不是root的话是不具备root权限的，客户端所有非root都会被当成other。

上面的选项中有个async，它表示文件是异步写入NFS服务器的，这里之所以这么配置是因为windows的文件浏览器特别不好使，它在打开一个文件夹时会读取很多信息，导致在windows上挂载一个NFS文件夹使用起来非常难受。异步的话就可以将写入延后，稍微提升一些流畅性。默认是 Sync，表示必须写入NFS服务器，写的操作才能成功返回。

上面的配置完成之后，执行`exportfs -arv` 重新加载NFS目录，输出如下：
```bash
[root@itdocker influxdb]# exportfs -arv
exporting 172.31.73.76:/opt/docker_files
exporting 192.168.22.0/24:/opt/softwares
exporting 192.168.22.0/24:/opt/docker_data
exporting 192.168.22.0/24:/opt/docker_files
```
可以在本地看看有哪些可用的NFS文件夹，命令是`showmount -e localhost`，输出如下：
```bash
[root@itdocker influxdb]# showmount -e localhost
Export list for localhost:
/opt/softwares    192.168.22.0/24
/opt/docker_data  192.168.22.0/24
/opt/docker_files 192.168.22.0/24,172.31.73.76
```
一般配置没什么问题就能有上面的输出。

##客户端配置——linux
客户端只需安装rpcbind就可以了，安装完成之后，同样使用`showmount -e x.x.x.x`（NFS ip）查询有哪些可以挂载的文件夹。挂载的过程和挂载硬盘相似，首先创建一个挂载点，然后将NFS上的文件夹挂载在这个点上。参考的命令如下：
```bash
mkdir /opt/docker_files
mount 172.29.41.127:/opt/docker_files  /opt/docker_files
```
这样就挂载完毕了，可以随意创建一个文件实验一下，看一下NFS服务器和本地是否一致。

##客户端配置——windows
如果想要在windows上操作NFS文件，步骤稍微有一些麻烦。首先要开启NFS Client的功能，如下图
{% asset_img 2018-04-13_174116.jpg 在windows中开启NFS Client %}
开启完毕之后就可以使用形如右边的命令将NFS共享文件夹挂载在一个新的卷标上`mount 172.29.41.127:/opt/docker_files x:`。
```bash
C:\Users\admin>mount 172.29.41.127:/opt/docker_files x:
x: is now successfully connected to 172.29.41.127:/opt/docker_files

The command completed successfully.
```
这个命令是将/opt/docker_files的作为本地x盘使用，如下图：
{% asset_img 2018-04-13_174638.jpg windows挂载 %}

如果NFS服务器共享的文件是root文件，那么windows上将无法写入，因为windows使用这个目录时的身份不是root（windows没有和linux一样的用户机制）。为此要么将共享文件夹变成777，或者更改注册表。
解决办法就是让Win7在挂载NFS的时候将UID和GID改成0即可：打开注册表：HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\ClientForNFS\CurrentVersion\Default，增加两项：AnonymousUid，AnonymousGid,如图：
{% asset_img 2018-04-13_180050.jpg 修改注册表 %}
这样修改就会使windows用户以root身份登录，然后就可以有写的权限。

# 3.NFS使用的陷阱
1. [!!!重要]如果客户端没有断开连接，NFS服务器是关不了机的，除非kill掉rpcbind和NFS进程。所以常常会给NFS客户端配置autofs，autofs会在不使用这个目录的时候自动取消挂载。
2. linux 挂载一定记得设置开机挂载，在/etc/rc.d/rc.local中设置就行
3. 一定要注意NFS的版本，V2和V3是不支持文件锁的，必须手动在服务器端和客户端开启nfslock才行。而V4是内置的。
4. v2-3默认是异步写，而v4默认是同步写，所以一定要注意NFS版本。在客户端df -hT查看文件系统就能知道是V4 还是V2-3，V4显示为`NFS4`。一般来说centos6 系列是3，而centos7 系统是4。
5. 如果NFS服务器挂掉了，而客户端尝试使用这个挂载目录的话，会将命令直接卡死，所以NFS服务器挂掉的话对客户端影响很大。 
