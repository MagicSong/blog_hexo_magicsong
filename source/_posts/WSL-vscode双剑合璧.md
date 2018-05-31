---
title: WSL+vscode双剑合璧
date: 2018-04-20 17:10:19
tags: 
    - WSL
    - vs code
    - docker
    - k8s
catagories:
    个人心得
---

> 今天下午捣鼓了一下WSL，通过一些配置将WSL改造了一下，结合vs code之后，windows系统看上去越来越像一个硬盘了。下面记录下我的操作。😈😈😈😈😈😈😈😈😈😈😈😈😈

<!--more-->
公司给的电脑很卡，同时开WSL+docker+vscode+chrome+nodejs就已经卡的不行了，所以必须改造一下。按照我之前的[文章](https://doc.itdocker.rd.tp-link.net/2018/04/19/%E5%9C%A8windows%E4%B8%8B%E5%81%9A%E7%B3%BB%E7%BB%9F%E8%BF%90%E7%BB%B4%E2%80%94%E4%B8%AA%E4%BA%BA%E5%BF%83%E5%BE%97/#more)配置好WSL之后，就需要在WSL装一些软件了。由于我的工作内容主要有三部分：
1. 文档
2. docker
3. k8s

下面针对上面三个部分依依说明。
# 文档
我写文档用的markdown，word由于不支持语法高亮早早就被我抛弃了。markdown的框架用的是hexo，一个用nodejs写的静态博客框架。所以首先要在WSL中安装nodejs，记得不要用apt-get 安装，一大堆问题。直接从官网上拉编译好的二进制文件，`ln`到bin目录即可。需要`ln`的二进制文件主要是`node`和`npm`。在WSL中执行检查一下，输出如下：
```bash
/mnt/d/blog$ node --version
v8.9.1
/mnt/d/blog$ npm --version
5.5.1
```
Very Good。下面安装hexo，由于我司的服务器都不能上网，所以很多工作都是在离线环境下完成的。现在WSL是在windows下，终于有一个可以上网的linux啦！！ 🤣🤣🤣🤣🤣🤣🤣🤣🤣🤣。执行下面的命令安装**hexo**:
```bash
sudo npm install -g hexo-cli 
```
安装完之后进入我的博客文件夹，输入npm install即可。这样一来文档就配置好了。但是这里有一个瑕疵，由于SWL的限制，hexo的**SFTP**用不了，其实也无所谓，因为后来发现根本用不到，我用了更好的**rsync**。输入`hexo list post`看一下吧。{% asset_img 2018-04-20_173054.jpg hexo安装完成 %}
安装完成之后还需要设置SSH免密登录，步骤是先在WSL`ssh-keygen -t rsa`生成公钥和密钥，然后将公钥传输（追加，不是覆盖）到服务器的`~/.ssh/authorized_keys`文件中，这样就完成了免密登录。

# docker
我原本是在windows上安装docker的，主要目的有两个，一是去拉墙外的镜像，二是为了本地开发。去拉墙外的镜像现在已经通过一些途径解决，后面的就开始考虑用WSL代替。我的服务器是安装了docker的，现在就是要实现远程访问docker。首先要在服务器上开启这个功能，很简单，在docker.service中启动参数中添加`-H 0.0.0.0:2375`即可，添加后执行下面的命令重启docker：
```
systemctl daemon-reload
systemctl restart docker
```
重启之后就可以看到有docker进程在监听2375端口啦。接着从docker网站上下编译好的docker包，地址在→_→：https://download.docker.com/linux/static/stable/x86_64/。下载完成之后在WSL中解压缩，然后将其中的一个名称为docker的可执行文件拖到bin目录下即可，这是docker客户端，其实就是将命令转换restful API然后将结果呈现在命令行的工具。其余的二进制目前还用不到，可以先放着。客户端好了还需要配置远程服务器，非常简单。下面一句话：
```bash
export DOCKER_HOST=X.X.X.X:2375
```
XXXX是远程服务器的地址。把这个放入.profile文件即可。`source`一下.profile文件，看一下`docker info`的执行结果吧！ 🤭🤭🤭🤭🤭🤭🤭🤭🤭🤭🤭🤭🤭🤭
{% asset_img 2018-04-20_174901.jpg docker info结果 %}

# kubernetes
需要访问k8s集群的话需要`kubectl`，基于我们已经安装好了docker，可以直接使用容器中的kubectl作为命令直接使用（相比原生命令慢不了多少的）。这里继续记录一下如何本地安装**kubectl**。和docker一样，直接从官网上下载编译好的二进制文件即可，然后+x就可以了。下面汇总一下命令：
```bash
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.10.0/bin/linux/amd64/kubectl
chmod +x kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
```
客户端安装好之后，配置一下k8s服务器，很简单，只需要将k8s集群上的`admin.conf`拖到本地的`.kube/config`即可。输入`kubectl cluster-info`看一下吧。{% asset_img 2018-04-20_175918.jpg kubectl命令 %}

# 后续
一下午，终于把自己的工作环境弄的非常舒服了，现在基本上windows做系统运维都是用linux的操作了，装windows也基本上为一些个别需求了，windows上的docker也可以安静地躺在硬盘里，而不继续消耗我的内存了！
