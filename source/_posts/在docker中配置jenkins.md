---
title: 在docker中配置jenkins自动化部署
date: 2018-03-15 16:51:31
author: 宋雪涛
categories: docker
tags: 
    - CI
    - 自动化部署
    - docker
    - jenkins 
---
# 在docker中配置Jenkins
## Jenkins介绍
>Jenkins主要用于持续集成（Continuous Integration，CI），CI是现代软件开发领域的基石，它改变了团队对于整个开发过程的理解。一个好的CI架构能够使得从开发到部署顺序进行，更快地发现和修复bug,最终给客户带来更多的价值。每个专业的开发团队，无论大还是小都应该采用CI。CI的软件很多，我司采用了jenkins这个软件，下面主要介绍这个软件的在Docker中的搭建和配置。

Jenkins起源于Hudson,占据了很大的市场份额，可被各种大小的团队和用不同语言(.NET,Ruby,Groovy,Grails,PHP等)开发的项目使用。优点如下：

- 易用性；
- 可扩展。插件覆盖了版本控制系统，构建工具，代码质量度量工具，通知工具，和其它外部系统进行集成，UI自定义等；
- 活跃的社区；
- 稳定版本的支持。Long-term Support(LTS)发布。

除了上述优点，有了docker的加持，Jenkins更是像神灵附体，CI能力更上一层楼。我在本次调研中主要感受到了以下几个优点：
<!-- more -->
- 极速安装，极速部署，不需要知道jenkins依赖什么，如何安装，甚至不需要知道jenkins是干嘛的，在docker中pull一下就能打开jenkins主页，非常舒畅
- 部署测试需要用的工具例如maven,nodejs不需要事先为其搭建，需要用到的时候只要从docker中pull一个即可。
- 代码直接部署到一个全新的环境中，基本上很多软件不需要配置文件（端口之类的都不用改），非常舒畅

**由于本次调研只是初步尝鲜，对于利用docker实现CI的局限性了解还不够深入，需要后续观察。**

***

## Jenkins的安装与环境配置

### 1. Jenkins的安装

由于网络原因，本次Jenkins安装是在离线状态下进行的，所以docker也要从私有仓库上拉，安装过程没有可以联网的情况下那么方便，所以就采用了在有网的环境中部署，然后复制到内网环境中。我的PC是上有网，所以在PC上装了一个docker(强烈推荐像我一样的新手这么做，不仅有网方便，其次在PC上开发比linux上开发舒服多了)。在PC上安装docker之前，也强烈推荐将操作系统升级到windows 10，主要是因为windows10的虚拟化功能。在其他版本的操作系统上需要安装一个虚拟机，然后在虚拟机里运行docker，相比windows10的原生支持虚拟要差很多。**当然能有一个直接联网的linux服务器就更好了。**

在windows 10和linux中运行Jenkins的命令差不多，需要注意的是，jenkins官方镜像是**jenkins/jenkins**而不是**jenkins**(一般官方镜像是没有前缀的，但是jenkins不是)，这个镜像不包含一些额外的插件。jenkins教程中推荐的镜像是**jenkinsci/blueocean**，它集成了一个叫blue occean的插件，这个插件在UI上做了很多的改进，相比原来的UI，好看不少。除此之外，blueoccean的配置也比较简单（如果能够联网的话），所以本次实验使用的是blueocean版本的jenkins。输入下面的命令运行jenkins：
```bat
docker run ^
  --rm ^
  -u root ^
  -p 8080:8080 ^
  -v G:\DockerData\jenkins:/var/jenkins_home ^
  -v /var/run/docker.sock:/var/run/docker.sock ^
  -v "%HOMEPATH%":/home ^
  --network IT_DOCKER_NET ^
  --name Jenkins ^
  itdocker.rd.tp-link.net:5000/jenkinsci/blueocean
```
在linux中的命令类似，改变一下数据卷路径即可。
```bash
docker run \
  -u root \
  --rm \  
  -d \ 
  -p 8080:8080 \ 
  -p 50000:50000 \ 
  -v /opt/docker_data/jenkins_data:/var/jenkins_home \ 
  -v /var/run/docker.sock:/var/run/docker.sock \
  --network IT_DOCKER_NET \
  --name Jenkins \
  rdsource.tp-link.net:8088/jenkinsci/blueocean 
```
执行上述命令之后，就可以通过本地的8080端口访问了。上述命令需要说明就是`-v /var/run/docker.sock:/var/run/docker.sock`这个命令，这个命令可以实现在jenkins这个容器中创建其他容器，即*docker-in-docker*，具体的原理可以参考[About /var/run/docker.sock](https://medium.com/lucjuggery/about-var-run-docker-sock-3bfd276e12fd)，了解这个原理之后就可以感叹linux everything is file的理念。
***
### 2.jenkins的配置

* 插件配置

等待docker拉取镜像完毕，在浏览器中输入相应的地址和端口，就可以进入jenkins初始化界面。最初进入了界面时需要一个密钥，密钥在启动日志中，可以输入`docker container logs`查询刚才容器的日志，里面提到了这个密码。输入这个密码之后就是安装插件，选择安装推荐的插件，推荐的插件不一定都能安装上，如下图：

{% asset_img plugins_install.jpg) 推荐插件无法安装截图 %}
如果遇到这种情况，首先先选择跳过，然后进入jenkins页面中，点击如下路径：**系统管理**->**管理插**件->**高级**，在下图位置修改URL，
{% asset_img 2018-03-16_095736.jpg "更改update_center URL" %}
造成插件无法安装的主要原因还是因为一些不可描述的原因。可以在下面的链接中选择合适的镜像源，[jenkins镜像源](http://mirrors.jenkins-ci.org/status.html),之后需要手动安装那些没有安装上的插件。安装就在插件管理中安装即可，这里就不多赘述了。除了上述推荐的插件，还有一个**Gerrit Trigger**的插件需要安装，这个插件主要用于和公司gerrit集成。记得安装完所有插件之后重启jenkins。
* git配置

jenkins需要和gerrit集成，所以首先需要配置git，git的配置和以前是相同的，进入jenkins的bash中（如何进入可以参考以前的文档），按照下面的步骤配置即可：
1. 生成公私密钥ssh-keygen -t rsa -C songxuetao@tp-link.com.cn
   之后多次输入回车默认即可。当然可以输入密码保护自己的密钥。
2. 复制公钥到gerrit
   将~.ssh/id_rsa.pub中公钥复制到gerrit中
3. 添加config文件
   vi ~/.ssh/config,输入下面的信息保存即可（*下面的信息是告诉git如何使用公钥密钥，由于我司的gerrit较古老，还在用较不安全的diffie-hellman-group1-sha1加密算法，所以必须在git中配置下面的信息，不然git默认不会使用这种算法*）
```
Host *
KexAlgorithms +diffie-hellman-group1-sha1
```
配置完上述git之后，就可以让jenkins直接拉取gerrit的代码了。

* gerrit trigger配置

点击**系统配置**，在最后找到**Gerrit Trigger**配置项，在其中配置gerrit服务器。进入页面之后在左上角选择**Add New Server**，输入名称之后，勾选默认，然后进入配置界面。配置界面主要配置gerrit服务器的IP地址端口以及SSH传输数据用到的密钥和公钥等，最终配置如下图：
{% asset_img 2018-03-16_114753.jpg gerrit %}
如果gerrit配置的SSH端口不是8000，那么需要修改上述的端口，配置完成之后点击Test Connection测试一下，看到success之后就大功告成了。

* 其他配置

如果想要让Jenkins能够发送邮件，还需要做一些配置。点击**系统配置**，找到下面的选项——**Extended E-mail Notification**：
点击高级，输入相关的邮箱设置，包括服务器端口，发件人密码等等（大部分默认即可），一些高级特性目前还用不上，最终效果如下：
{% asset_img 2018-03-16_113413.jpg 配置邮箱 %}

如果要启用LDAP验证（我使用了一下LDAP验证，效果不好，无法实现访问控制，要实现需要从LDAP着手解决，比较麻烦。所以最终没有启用，选择了手动管理用户），也同样在这个页面中找到下面的配置项，输入相关信息即可。配置完成之后还需要在安全设置中启用LDAP，启用之后我司员工就能登录了。
{% asset_img 2018-03-16_114124.jpg 配置LDAP %}

***
## 3.jenkins使用入门介绍
>以前我司的jenkins的自动部署用到了很多SHELL脚本，并且脚本的执行很依赖于环境（即换个地方就很难运行起来了），此次引入了docker进行自动化部署，相比以往的自动化部署稍微要复杂一些（原因是我们的Jenkins部署在Docker中，无法直接运行宿主机中的程序），需要写一些脚本，增加了学习成本。此次之所以用脚本的原因在于脚本的可扩展性高，且易于维护，一旦jenkins崩溃了不会影响自动化部署流程。Jenkins用的脚本是一个名为jenkinsfile的文件（没有任何后缀，和dockerfile一样），在这个脚本写一些代码，Jenkins就会按照脚本中写的步骤依次执行。下面演示以SMBCommunity这个项目为例，做一个说明。

SMBCommunity的架构是Java+ExtJS+Spring，项目用maven构建，编译整个项目需要用到maven和sencha，Web服务器用的则是开源的Tomcat。如果不做测试的话，整个自动化的流程如下图：
{% asset_img 2018-03-16_155220.jpg 项目自动化流程 %}
同时构建触发可以是手动触发，也可以是代码触发，即这个项目里有了新的代码，那么Jenkins就会为它重新构建一个Tomcat应用。
### blue ocean介绍
相比以前，**blue ocean** 主要是在UI上做了很大的改进，在视觉体验上更现代化，也更简洁大方，下图是blue ocean首页效果。
{% asset_img 2018-03-17_100752.jpg "blue ocean首页" %}
除了在UI上有了很大的改进，blue occean也支持在线编辑pipeline，同时原生支持Github的pull request。相比于Jenkins经典UI，blue ocean对构建过程的可视化效果也要好很多，不同的背景代表了构建的结果，比如红色代表失败，绿色代表成功。
### 操作流程
下面就按照这个流程在Jenkins中设置。为了演示方便，我需要添加一些新的文件，我Fork了一份代码，在Gerrit中获取其SSH路径。然后在Jenkins中选择新建任务，在任务类型中选择**流水线**（pipeline）,如下图
{% asset_img 2018-03-16_160134.jpg 新建一个流水线 %}
在打开的配置中，有几个配置要注意，第一个是构建触发器，选择Gerrit event，也可以选择周期性构建，但是周期性构建有一点浪费资源。选择Gerrit event之后会展开一些选项，在其中选择我们上面设置的Gerrit服务器，Trigger on 选择**Change Merged**，Gerrit项目中填写相关的项目名称和分支，这部分最终填完是下图的样子。
{% asset_img 2018-03-16_161631.jpg "Gerrit Trigger设置" %}
在流水线中选择Pipeline script from SCM，这个选项意思是jenkinsfile从源码中读取，所以在我们Fork的源码根目录下我们需要添加一个文件叫jenkinsfile。选择这个项目之后就会展开让你选择Git的URL（如果不选择这个选项的话，需要自己写脚本从某个URL拉取代码，比较麻烦），SCM选择Git，将从Gerrit上拷贝的URL复制到Repository URL，输入想要Build的分支，最终填写如下：
{% asset_img 2018-03-16_162348.jpg 流水线设置 %}
上面几个都填写完毕之后保存即可。这样一来，每当Gerrit有代码通过审核之后，Jenkins就会启动一次构建。如何构建是写在jenkinsfile中的。
### 撰写jenkinsfile
在源代码中创建一个Jenkinsfile，有下面这些好处：
1. Pipeline上的代码审查/迭代
2. Pipeline的审计跟踪
3. Pipeline的唯一真实来源，可以由项目的多个成员查看和编辑。

Jenkinsfile实际上对应的就是一个Pipeline脚本,Pipeline支持两种语法：**Declarative**（在Pipeline 2.5中引入）和**Scripted** Pipeline。两者都支持建立一个连续流程Pipeline。两者都可以用于在Web UI或者源代码中中定义一个流水线Jenkinsfile，但Jenkins推荐的做法也是将Jenkinsfile放入源码中，而不是直接在Jenkins中设置。上面提到的两种语法中，学习难度较小的是Declarative，本文也是用的Declarative，Scripted Pipeline是jenkins的基础，但是容易度和可阅读性上都比Declarative差一点，所以我没有学Scripted的相关语法，从学习结果看，Declarative语法已经足够我们开发人员使用了。下面的代码如果不加说明都是Declarative语法。

#### 创建Jenkins文件
 Jenkinsfile是一个包含Jenkins Pipeline定义的文本文件，并被检入源代码控制，pipeline中每一个步骤的属性时通过括号和缩进控制的。下面贴上的代码是一个包含三个顺序执行过程的Pipeline。
``` 
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                echo 'Building..'
            }
        }
        stage('Test') {
            steps {
                echo 'Testing..'
            }
        }
        stage('Deploy') {
            steps {
                echo 'Deploying....'
            }
        }
    }
}
```
pipeline是最外层元素，表示这是一个Declarative 语法pipeline，它的括号里面是关于这个pipeline具体执行过程的信息。`agent`表示这些步骤在哪些环境中执行，通常有两种any和docker，any一般都是直接在jenkins中执行，而docker则是生成一个容器执行命令，本次实验中一部分命令需要在docker中执行。`stages`表示一系列执行环节，从上面的代码中可以看到它是由一些stage组成的（上面代码中包括了Build，TEST，Deploy三个环节）。另外`agent`元素和`stages`元素表明这些环节默认的执行环境都是any，当然也可以在`stages`的子元素`stage`中再定义的一个agent，这个agent会覆盖默认的设置生效。在一个步骤`stage`中，会包含一些步骤`steps`,这些步骤会按照书写的顺序依次执行。上面的步骤只是简单的输出几个语句。
将上面的代码复制到项目源代码根目录下的jenkinsfile中，然后推送到gerrit上评审。commit消息写 Add Jenkinsfile。打开Gerrit中，将上述commit的评审通过。一旦评审通过，立刻打开jenkins，在右边选择open blueocean，就可以看到如下消息：
{% asset_img 2018-03-16_173202.jpg 自动触发构建 %}
从图上看，一旦评审通过，那么gerrit就会自动构建一次。点开这次构建查看具体信息。如下图：
{% asset_img 2018-03-16_173417.jpg 构建成功截图 %}
绿色的背景代表了构建是成功的，具体构建的信息可以在Log中找到。至此，在jenkins中自动化部署的简单示例就完成了，接下来就需要完善Jenkinsfile，使其能够真真满足实际工作需要。关于Jenkinsfile脚本的语法可以在[官网链接](https://jenkins.io/doc/book/pipeline/)上找到，稍微熟悉一下脚本之后，可以使用jenkins主页上的片段生成器生成相应的代码，参考链接：[片段生成器](http://itdocker.rd.tp-link.net:9001/job/Jenkins_Test/pipeline-syntax/)。
