---
title: ITDocker资料清单
date: 2018-03-17 10:18:48
author: ADMIN
thumbnail: /img/wallhaven-19194.png
sticky: 1
categories:
    - 系统管理
tags:
    - system
    - docker
---

# ITDocker端口使用清单

| 端口        | 服务                          |
| ----------- | ----------------------------- |
| 5000        | IT课Docker私有Registry        |
| 5001        | IT课Docker私有Registry可视化工具 |
| 9000        | portainer，Docker管理程序     |
| 9001        | Jenkins，CI服务               |
| 9002        | ITDocker文档整理              |
| 26379-26381 | redis_sentinel，一共有3个哨兵 |
| 50000       | Jenkins 安全管理Agent端口     |

请使用**itdocker.rd.tp-link.net:PORT**访问相关应用，开发人员用于开发和测试端口一般在**8000-8999**之间，这些端口全公司都可以访问。
<!-- more -->
# ITDocker自制镜像一览
> 具体这些镜像的信息请在[私有镜像管理器](itdocker.rd.tp-link.net:5001)查看

| 镜像名称                                           | 特性                                                         |
| -------------------------------------------------- | ------------------------------------------------------------ |
| itdocker.rd.tp-link.net:5000/itmaven:sencha        | 集成了sencha，并且配置了公司内网MAVEN仓库                    |
| itdocker.rd.tp-link.net:5000/itjenkins:latest      | 集成了Docker-Compose                                         |
| itdocker.rd.tp-link.net:5000/ittomcat:latest       | 修改了Server.XML，以便自动部署                               |
| itdocker.rd.tp-link.net:5000/itmaven:latest        | 不包括sencha的轻量级maven，配置了公司内网MAVEN仓库           |
| itdocker.rd.tp-link.net:5000/redis_sentinel:alpine | 自制redis_sentinel，主要用于Docker-Compose，不适合直接生成容器 |

