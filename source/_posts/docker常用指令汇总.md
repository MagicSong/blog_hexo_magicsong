---
title: docker常用指令汇总
date: 2018-03-19 10:33:55
tags:
---
> 在这里记录一些docker常用指令，便于后续使用时查询

# docker相关
## 1. 系统使用
```bash
#开机启动docker
systemctl enable docker
#启动docker
systemctl start docker
```
<!-- more -->
## 2. docker容器相关操作
> docker容器可以理解为在沙盒中运行的进程。这个沙盒包含了该进程运行所必须的资源，包括文件系统、系统类库、shell 环境等等。但这个沙盒默认是不会运行任何程序的。你需要在沙盒中运行一个进程来启动某一个容器。这个进程是该容器的唯一进程，所以当该进程结束的时候，容器也会完全的停止。

```bash
# 启动、停止等操作
docker start|stop|restart [id]
# 暂停|恢复 某一容器的所有进程
docker pause|unpause [id]
# 杀死一个或多个指定容器进程
docker kill -s KILL [id]
# 停止全部运行的容器
docker stop `docker ps -q`
# 杀掉全部运行的容器
docker kill -s KILL `docker ps -q`
# 查看当前运行的容器
docker ps
# 查看全部容器
docker ps -a
# 查看全部容器的id和信息
docker ps -a -q
# 查看一个正在运行容器进程，支持 ps 命令参数
docker top
# 查看容器的示例id
sudo docker inspect -f  '{{.Id}}' [id]
# 检查镜像或者容器的参数，默认返回 JSON 格式
docker inspect
# 返回 ubuntu:14.04  镜像的 docker 版本
docker inspect --format '{{.DockerVersion}}' ubuntu:14.04
# 创建一个容器命名为 test 使用镜像ubuntu
docker create -it --name test ubuntu
# 创建并启动一个容器 名为 test 使用镜像ubuntu
docker run --name test ubuntu
# 删除一个容器，删除之前需要先停止，或者也可以加-f参数强制停止
docker rm [容器id]
# 删除所有容器
docker rm `docker ps -a -q`
```
## 3. docker 镜像相关操作
```bash
# 根据Dockerfile 构建
docker build -t [image_name] [Dockerfile_path]
# 列出本地所有镜像
docker images ls 
# 本地镜像名为 ubuntu 的所有镜像
docker images ubuntu
# 查看指定镜像的创建历史
docker history [id]
# 本地移除一个或多个指定的镜像
docker rmi
# 移除本地全部镜像
docker rmi `docker images -a -q`
# 指定镜像保存成 tar 归档文件， docker load 的逆操作
docker save
# 删除所有标记为None的镜像
docker rmi $(docker images -f “dangling=true” -q)
# 将镜像 ubuntu:14.04 保存为 ubuntu14.04.tar 文件
docker save -o ubuntu14.04.tar ubuntu:14.04
# 从 tar 镜像归档中载入镜像， docker save 的逆操作
docker load
# 上面命令的意思是将 ubuntu14.04.tar 文件载入镜像中
docker load -i ubuntu14.04.tar
docker load < /home/save.tar
# 构建自己的镜像
docker build -t <镜像名> <Dockerfile路径>
docker build -t xx/tomcat .
```
# Docker-Compose相关
