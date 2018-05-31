---
title: 在windows下做系统运维—个人心得
date: 2018-04-19 09:36:12
tags: 
    - WSL
    - vs code
catagories:
    - 个人心得
---

> 随着学习的慢慢深入，越来越觉得在终端工作是一个很舒服的事情，虽然还是有很多工具离不开一些GUI，比如PLSQL，印象笔记，outlook等，但是这类的工具开始越来越少了。这里记录一下我的心得，记录一下我是如何打造一个属于我的工作环境的。

<!-- more -->
# 介绍
虽然很想将自己的操作系统直接变成linux，但是由于公司很多东西都很依赖windows，所以暂时还是无法离开windows的。但是windows的爸爸微软对待开源世界也越来越开放了，对linux的态度也越来越好，对linux的支持真是越来越棒了。感受最深的就是vs code和WSL（Windows Subsystem for Linux），尤其是后者，随着WSL的越来越成熟，在windows下使用linux变得越来越简单，开发linux脚本和其他脚本都可以直接在WSL中运行，运行成功了之后可以部署在真正的服务器中。非常的方便。

这里想说明一下为什么不直接在服务器中做开发呢？这个还是个人原因，公司的服务器是无法联网的，服务器中的vim是无法安装插件的（大多数也不会去安装插件），对于一个习惯了在windows下编程的人来说（语法高亮，缩进支持，snippets），很多功能都用不了，心里微微有点不舒服。个人是还不习惯用vim，而且自从有了vs code之后，完全没有使用vim的动力。所以研究了一套在windows下做系统运维的方法，使用时间不长，但是目前还是很舒服的。


# 安装必要工具
## windows 10
微软对开源世界的大力支持是从纳德拉开始的，纳德拉时代的操作系统是windows 10，windows 10之前操作系统是无法使用WSL的，所以下文所有的一切都是在windows 10下的。WSL刚出来的时候还不完善，随着一次一次的更新WSL变得越来越好用，所以使用windows10一定要保持系统日常更新。安装windows 10这里就不介绍了。

## Visual Studio Code
这个神器自从发布以来就越来越流行了。可以根据自己的需求随心所欲定制，想让它变成什么IDE就能变成什么IDE。目前我在VS Code上配置的环境能够用于写linux脚本，写markdown文章，写NodeJS程序。现阶段主要用于维护k8s集群。安装的话直接去首页吧[https://code.visualstudio.com/]

## WSL
很过工程师喜欢在MAC中工作就是因为MAC对linux终端支持是非常棒的，有了WSL之后，就可以在愉快在windows中敲代码，敲命令行，而不需要使用难用的CMD啦！Windows 10 中包含了一个 WSL（Windows Subsystem for Linux）子系统，我们可以在其中运行未经修改过的原生 Linux ELF 可执行文件。利用它我们可以做很多事情，对开发人员和普通用户都是如此。当然对开发人员的吸引力更大一些，因为这意味着在一些情况，不再需要使用 Linux 虚拟机、双系统、Cygwin/MSYS2 了。安装过程在这个网页中[https://docs.microsoft.com/en-us/windows/wsl/install-win10]，下面就是简单罗列一下：

以**管理员身份**运行powershell，输入下面的命令：
```powershell
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Windows-Subsystem-Linux
```
+ 这个步骤之后需要重启，重启之后打开windows store中查找自己要安装的系统，目前支持下面几个：
+ Ubuntu
+ OpenSUSE
+ SLES
+ Kali Linux
+ Debian GNU/Linux
我选择的ubuntu，因为它使用的比较广,而且最关键的是vs code原生支持（vs code的最新版本，我使用的1.22.2是原生支持的）。什么系统确实无所谓的，我们只要用的还是其中的bash。选择好之后就可以安装了，安装过程基本不需要人工干预，一般只需要输入用户名和密码即可。安装完成之后在CMD中输入`bash`就可以进入WSL的ubuntu 了（不同的系统指令不一样，ubuntu是`bash`），或者也可从开始菜单中点击ubuntu的进入，如下图：
{% asset_img 2018-04-19_101617.jpg 开始菜单中的ubuntu %}
进入之后的页面如下：
{% asset_img 2018-04-19_101745.jpg WSL界面 %}
在vscode中是这样的：
{% asset_img 2018-04-19_101830.jpg WSL in vscode %}

WSL和windows系统是互不干预的，windows中的磁盘是通过挂载的方式提供给WSL访问的。挂载的位置在`/mnt`中，其中可以看到我们熟悉的CDE盘。如下：
```bash
magicsong@W10405:~$ ll /mnt
total 0
drwxr-xr-x 0 root root 4096 Jan 31 08:37 ./
drwxr-xr-x 0 root root 4096 Jan  1  1970 ../
drwxrwxrwx 0 root root 4096 Apr 18 10:40 c/
drwxrwxrwx 0 root root 4096 Apr 18 11:08 d/
drwxrwxrwx 0 root root 4096 Apr 18 16:18 e/
drwxrwxrwx 0 root root 4096 Apr 18 11:03 f/
drwxrwxrwx 0 root root 4096 Apr 18 11:03 g/
```
虽然很多内核功能无法实现，WSL用于文字处理工作是极其胜任的了，尽情在windows中使用`grep`，`awk`和管道处理windows文件吧！😎😎😎😎😎😎😎😎，你会发现用起来非常爽，而且支持auto competition哦！

## 配置工作
安装好必须的软件之后，就需要做一些配置工作。具体用到有下面几个，大家可以参考一下：
1. 更改默认字体（个人主观需求）。默认字体中的中文显示真是非常不符合我的审美。当然由于控制台比较特殊，它的字体渲染和一般的GUI不一样的，所以改字体的难度很大（大部分中文字体是无法在控制台下使用的，要么用起来极其难看）。不要怕，有大神教——[为什么 Windows 下 cmd 和 PowerShell 不能方便地自定义字体？ - Belleve的回答 - 知乎](https://www.zhihu.com/question/36344262/answer/67191917)，改完之后的bash长这样：{% asset_img 2018-04-19_103923.jpg 改为后的字体 %}
2. 卸载掉所有IDE，文本编辑器（sublime，notepad++等等），除了数据库相关的。
3. 部署文件同步工具，推荐使用SFTP，在本地测试完成之后，直接SFTP同步到服务器上。我用的是**freefilesync**，可以本地同步，也可以和服务器通过SFTP同步，非常方便。这个工具可以脚本化，从而实现自动同步。
4. 如果服务器是集群，可以使用NFS的方式，集群之间通过NFS同步代码文件，windows本地可以选择直接挂载NFS文件（这个方式不推荐，用起来不舒服。主要原因还是windows的文件目录系统和linux不一致。linux用到时才会查询文件信息，而windows未使用前就要获取文件信息。NFS出于性能原因时不时会断开，windows每次断开重连之后就要同步，所以会很卡）。
5. 在vs code安装一些必要的插件，根据自己需要选择安装。这里罗列一些我强烈推荐的：
    + Markdown All in One ，更方便写文章
    + Material Icon Theme，给文件添加好看的图标
    + Path Intellisense，写代码时方便插入路径
    + Bracelet Pair Colorizer，给配对的括号涂上不同的颜色，这样就不会搞混括号啦！
6. 在WSL安装kubectl，安装之后就可以在WSL维护k8s集群了，具体安装参考[在ubuntu中安装kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-kubectl-binary-via-native-package-management)。安装完成之后就可以愉快在WSL中操作k8s了(windows下也是可以用的。但是CMD的操作和linux 差距太大，实在是用不惯，而且没有`grep`,`sed`,`awk`，你能想象一下如何使用！)。如下图：{% asset_img 2018-04-19_113507.jpg 在WSL中操作k8s %}


# 后续
后面如果有什么好的配置会在这里继续更新。**我的目标是一个vs code，一个浏览器，我就能随时随地高效工作！**