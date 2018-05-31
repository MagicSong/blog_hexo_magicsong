---
title: 如何在k8s中部署一个web应用
date: 2018-04-19 17:17:18
tags: 
    - k8s
    - 部署
    - http
catagories:
    - docker
    - kubernetes
---
> 这篇文章主要论述如何在k8s中部署一个应用，并且符合k8s规范。具体部署的内容因不同的应用而异，这里会罗列一些注意事项。
<!--more-->

# 部署应用的流程

## 什么是服务
kubernetes中的最小单位是POD，POD往往是一个和几个关联的容器（`container`）。原生docker往往直接暴露的就是容器，应用通过端口直接访问容器，但是这种方式不利于负载均衡和高可用，因为一旦这个容器挂掉了或者IP改了，都会造成应用不可用。k8s中是以service作为应用访问的起点，如下图：{% asset_img services-iptables-overview.svg k8s中的服务模型 %}

下面贴出了一个样例服务，来自于doc.itdocker.rd.tp-link.net上应用。
```yaml
kind: Service
apiVersion: v1
metadata:
  name:  docker-docs
spec:
  selector:
    app:  docker-docs
  ports:
  - name:  http
    port:  80
    targetPort:  80
```
上面的服务非常简单，因为大部分是默认属性了，这里论述一些关键属性。第一个是`selector`，`selector`表示选择这个service背后的实际后端的逻辑。这里的意思就是这个service选择后端具有app: docker-docs的pod，任何在这个命名空间里具有这个label的都会被选择。`ports`表示后端pod暴露的端口和自身端口的映射，其中`port`是服务自身暴露给外界的端口，`targetPort`就是后端pod的端口。这样一个服务就定义好了，它自身暴露了80端口，然后把经过自身80端口的流量转发给后端的POD处理（默认选择的方式是round-robin）。**这里有个注意点**，虽然service能够把流量转发到后端POD，但是当我们按照此文真正部署完一个应用之后，实际的流量是通过nginx控制的（名称叫ingress），service只是起一个后端发现的作用。当然还有一些其他的部署方式（不走本文的方法，用proxy，用nodePort），它们的流量是过service转发的。

## Deployment
写好service之后就要写真正处理逻辑的后端，后端可以直接写`POD`，也可以用`Deployment`，本文推荐用`Deployment`，`Deployment`可以控制应用的可用数量，并且控制如何在集群节点上分布我们的应用。下面贴上一个我写的Deployment。
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: docker-docs
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: docker-docs
        editor: vscode
    spec:
      imagePullSecrets:
        - name: pull-secret
      containers:
        - name: docker-docs
          image: rdsource.tp-link.net:8088/nginx:alpine
          ports:
          - name:  http
            containerPort:  80
            protocol: TCP
          volumeMounts:
            - name:  document-storage
              mountPath:  /usr/share/nginx/html
      volumes:
        - name:  document-storage
          hostPath:
            path:  /opt/docker_files/documents   
```
上面这个的Deployment简单部署了一个nginx应用（读取静态HTML并显示出来）。可以看到replica是2，说明会在集群中部署两个POD，作为一个简单的负载均衡和高可用。POD通过`containerPort`属性暴露了80端口，这个和上面的service中的`targetPort`是一致的，不一致会导致服务不可用。Deployment指定了需要的镜像和镜像名称。这里也有一个注意事项，就是私有仓库拉取。我司的私有仓库是一个代理，拉取镜像要有用户名和密码。在主机上操作一次时，只要用`docker login`一次，以后就不用再登录了，具体的登录信息会保存在`~/.docker/config.json`中。这个登录信息是会过期的，我在测试的时候一般是重启之后就会过期，需要重新登录。但是k8s不会使用上述登录信息，所以每次从私有仓库拉取镜像（如果需要用户名和密码）都必须指定secret，上面语句中的`imagePullSecrets`就是这个secret。secret的生成模板如下：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name:  pull-secret
data:
  .dockerconfigjson: XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
type: kubernetes.io/dockerconfigjson
```
其中的` .dockerconfigjson`就是`cat ~/.docker/config.json`的输出结果。需要注意的是secret不能在不同的namespaces中共享，需要为每一个namespace建立一个secret（这个有点设计不合理了，我觉得）。

## 访问服务
至此，一个服务和它的后端都建立起来了。但是外界是无法访问这个服务的，而集群内部可以通过[serviceName].[namespace]的格式访问。大部分服务都是要提供给外界的，那么如果访问呢？有多种方式：
1. 在service中配置
    + 将service的type设置为`nodePort`，那么系统会随机为服务分配一个30000以上的端口号，然后通过任意节点+这个端口号进行访问。缺点很明显，端口号很大，不容易记住，而且是随机的。
    + 将service的type设置为`loadBalancer`，让集群的云服务商提供一个外部ip作为VIP访问这个服务。这是在云服务中通用的，但是在私有集群中难以实现。
    + 用`kubectl proxy`做代理，访问服务。缺点很明显，每次访问都要做代理，而且只有执行`kubectl proxy`的那个主机才能访问。但也有优点，非常安全。对于一些安全级别很高的应用，应当使用这个方式。
2. 用ingress
**ingress**表示了一个应用的入口配置，它是通过`ingress controller`实现的。**ingress controller**的简介如下：
> Configuring a webserver or loadbalancer is harder than it should be. Most webserver configuration files are very similar. There are some applications that have weird little quirks that tend to throw a wrench in things, but for the most part you can apply the same logic to them and achieve a desired result.  
>The Ingress resource embodies this idea, and an Ingress controller is meant to handle all the quirks associated with a specific "class" of Ingress.  
>An Ingress Controller is a daemon, deployed as a Kubernetes Pod, that watches the apiserver's /ingresses endpoint for updates to the Ingress resource. Its job is to satisfy requests for Ingresses.

ingress controller主要作用就是负载均衡和web应用配置，ingress controller本身也是一个服务，它将自身的特定端口暴露在外部世界中，外部通过域名访问时，ingress controller会通过域名规则将流量转发到指定的后端（Endpoints）。这里有一个小知识点，ingress controller是直接转发到后端POD上的，而不是转到服务上，大家知道服务是round-robin的方式选择后端的，而ingresscontroller可以实现更加灵活的选择方式，比如权重啊等等。看到这，大家肯定有疑问，这不就是**nginx**吗？哈哈，就是**nginx**实现的模式，k8s官方推荐的也是用nginx加工的，相比原生nginx，ingress controller最大的特点就是nginx的配置文件自动生成，我们不需要为所有的应用去写nginx语法，仅仅只需要一个ingress文件即可。所以要想使用ingress，就必须先安装ingress controller。安装过程见我其他文章（TODO）。
下面贴上我们的ingress:
```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
    kubernetes.io/tls-acme: 'true'
  name: docker-docs
spec:
  tls:
  - hosts:
    - doc.itdocker.rd.tp-link.net
    secretName: itdocker-tls
  rules:
  - host: doc.itdocker.rd.tp-link.net
    http:
      paths:
      - backend:
          serviceName: docker-docs
          servicePort: 80
        path: /
```
上述ingress是一个加持了https的配置，其中`host`和`hosts`属性指定了访问的域名，即用户通过`doc.itdocker.rd.tp-link.net`这个域名访问，ingress controller就能识别用户是想要访问后端的一个docker-docs的服务。这里虽然写了服务，但是实际流量是直接转发到后端POD上的哦，服务只是作为一个查询中介（这个小原理不知道也无所谓 😇😇😇😇😇😇）。`path`可以指定域名后的路径，我在使用后面路径的时候常常遇到404的问题，主要是因为用户通过带路径的域名访问时，一些应用的一些资源是用的绝对路径，比如我们ingress指定了`docker.net/a`转发到a应用，但是a应用中某个资源路径是从根目录开始的，比如/api/XXX，a应用这么请求肯定找不到资源，因为按照我们的ingress设置，路径应当为/a/api/XXX才能正确转发。有些应用支持baseURL属性，有了baseURL，它的所有请求都是带baseURL的，这种应用就可以通过路径的方式区别开来。而其他不支持baseURL的应用在这种模式下很容易出错。无奈之下，我申请了域名的子域名通配符，这样一来，我就可以按照我的设计为docker上的应用配置ingress和其域名。
上述ingress使用了SSL证书，所以访问这个应用时会自动跳转到ingress controller的443端口，SSL证书和密钥存放在上述`itdocker-tls`这个密钥中。可以看到https加密是在前端nginx做的，而不是后端，后端还是80端口。为了用户能够正确访问，需要将域名绑定到集群中的任意一个节点，我绑定到了我的master节点。当然也可以搞一个虚拟IP。虚拟IP的方式比较合理，我还没有尝试，下次可以试试。

至此，将上述所有文件apply一下，一个完整的应用就好了。下图整理了本文的思路，具体的流程就如下：{% asset_img ingress.png ingress controller原理简图 %}

# 后续
ingress controller是部署在L7,但是很多应用需要在L4上部署（mysql,redis,git clone），如何利用ingress暴露TCP服务我还要继续研究一下。