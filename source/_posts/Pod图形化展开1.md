---
title: Pod图形化展开
index_img: /img/pod_1.png
banner_img: /img/bg.jpg
date: 2023-02-17 17:25:55
tag: 技术
---


## 背景

最近接触到Iscca gym,MuJoCo这类的物理引擎可以在linux中尝试实时渲染，但我们的镜像运行在云端kubernetes，没有可用的图形化界面，直接导出数据再渲染的方式会有一定的滞后性。如果既想使用我们性能强大的gpu进行运算，又想直观地看到实时反馈，唯一的解决办法就是打通Pod到本地的图形渲染管道。

## X协议

Linux本身是没有图形化界面的，所谓的图形化界面系统只不过中 Linux 下的应用程序。这一点和 Windows 不一样。Windows 从 Windows 95 开始，图形界面就直接在系统内核中实现了，是操作系统不可或缺的一部分。Linux 的图形化界面，底层都是基于 X 协议。

![img](/img/pod_1.png)

把X server和X client抽象成windows上的 主机 + 显示器 。

我们看到的显示器+键鼠就是X client ，供我们交互。而Pod上的linux系统就是X server 。

我们要做的是在Pod上开放x11 forwarding供本地连接。

## x11 forwarding

#### **远程服务器端**

在Linux打开x11 Fording:

```Bash
sudo vim /etc/ssh/sshd_config

/**************************/
# 配置转发参数（自己手动修改）
X11Forwarding yes
X11DisplayOffset 10
/****配置完毕后重启ssh服务****/

service ssh restart
```

#### 本地客户端

macOS安装**xquartz**：

```Bash
brew install --cask xquartz
```

安装好后位置比较隐蔽，第一次打开的时候找了很久，在应用程序 -> 实用工具（工具） -> xquartz

![img](![img](https://docimg1.docs.qq.com/image/AgAABTJcxKr7q-IkEe9KergOkMU4Q_MO.png?w=1374&h=184))

在打开的命令行窗口中输入ssh连接指令连接即可：

```Bash
ssh -X 你的名字@你的地址

（比如 ssh -X rakkanhong@someAddress）
```

windows安装**MobaXterm：**

MobaXterm 是一款开源免费的、全功能终端软件，自带 X Server。下载地址如下：

[MobaXterm free Xserver and tabbed SSH client for Windows](https://mobaxterm.mobatek.net/download.html)

安装成功后同样在终端输入ssh连接指令链接即可

#### 测试连接

连接成功之后，输入xclock测试：

- 如果X11 Forwarding没有成功开启的话，会显示 Error: Can't open display:
  - 
- 如果成功开启，会展示出一个时钟的页面
  - ![img](/img/pod_2.png)

看到这个界面就大功告成了

## Kubernetes开启ssh连接

正常能够通过ssh连接的服务器到上面已经结束了，但我们的Pod是通过kubectl exec的方式进入容器，并不是通过ssh进行远程访问的，这里有一篇关于两种不同连接方式的对比[SSH vs.kubectl exec_新钛云服的博客-CSDN博客](https://blog.csdn.net/NewTyun/article/details/108525876)。

由于这种方式实现的连接没有建立起ssh隧道，Pod中的DISPLAY仍然为空，即使是手动进行设置也没办法进行工作，所以想要尝试使用ssh的方式连入Pod。

实际的操作其实也并不复杂，需要首先要在Pod里安装开启ssh服务，接着另外增加一个用户用于连接（Pod里初始的root没有初始密码，而且增加了密码之后尝试也没有连接成功）。使用port-forward功能将Pod的22端口forward至本地指定端口，再通过ssh的方法进行连接：

```Bash
## port-forward
kubectl port-forward pods/jupyter-rakkanhong 10086:22 -n tensorpipes

##ssh
ssh -X testuser@localhost -p 10086
```

连通后输入netstat -nltp确实可以看到22端口已经开启监听，但查看DISPLAY参数仍然为空，尝试失败。

最终尝试了一下VNC Server 并且连接成功。

## VNC 

VNC (Virtual Network Console)是一款基于 UNIX 和 Linux 操作系统的优秀远程控制工具软件，由著名的 AT&T 的欧洲研究实验室开发，远程控制能力强大，高效实用，并且免费开源。

不同于使用X11 forwarding在连接成功后就能自动将X Client当做$DISPLAY来显示图像，VNC需要连接在服务器中现有的图形界面。所以我们需要借助到Xvfb和x11vnc来搭建虚拟X Server供我们本地连接。

#### **远程服务器端**

首先在一个终端中启动一个虚拟屏幕：

```Bash
Xvfb :1 -ac -screen 0 960x540x24
```

启动完毕后这个终端之后保持在后台不要关闭，另起一个终端运行x11vnc:

```Bash
x11vnc -display :1 
```

可以在终端中看到详细的log信息，同时可以看到5900的端口已经打开，需要将该端口forward到本地，在vscode中会自动进行Port-forward，如果需要手动forward的话需要在本地输入以下指令：

```Bash
kubectl port-forward pods/{pod名} {本地端口}:5900 -n {namespace名}

（比如 kubectl port-forward pods/jupyter-rakkanhong 10086:5900 -n tensorpipes )
```

当然如果不是在Pod里的话，直接使用服务器IP+端口就行了，连接起来比上面的ssh还要方便。

#### 本地客户端

Windows和macOS都可以用下述链接下载VNC Server

[Download VNC Server | VNC® Connect](https://www.realvnc.com/en/connect/download/vnc/)

安装后打开VNC Viewer，输入localhost:{本地端口}后就能跑通了，不过注意此时的连接方式是相当于投屏的方式进行展示。具体需要展示的东西需要在远程绑定了前面虚拟出来的DISPLAY后才能在本地看见。例如新建一个终端后输入：

```Bash
#设置DISPLAY,注意引号内的编号为之前xvfb所设置的编号
export DISPLAY=":1"
xclock
```

![img](/img/pod_3.png)

可以看到本地的Viewver已经成功显示出所连接的远程桌面了，自此成功打通Pod上的图形展开。

```Bash
apt-get install build-essential freeglut3 freeglut3-dev binutils-gold
```

## 参考链接

[通过X11实现 Linux服务器图形化界面显示 - 李晓春 - 博客园](https://www.cnblogs.com/lixiaochun/p/8547815.html#:~:text=X 协议由 X server 和 X client 组成：,client。 l X client (即 X 应用程序) 则主要负责事件的处理（即程序的逻辑）。)

[macOS 使用 X11 运行远端 linux 中的 x11 client 图形程序_mac x11_longyu_wlz的博客-CSDN博客](https://blog.csdn.net/Longyu_wlz/article/details/128068675?spm=1001.2101.3001.6650.1&utm_medium=distribute.pc_relevant.none-task-blog-2~default~AD_ESQUERY~yljh-1-128068675-blog-107871367.pc_relevant_aa2&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2~default~AD_ESQUERY~yljh-1-128068675-blog-107871367.pc_relevant_aa2&utm_relevant_index=2)

[如何从集群外部通过SSH进入Kubernetes Pod ？](https://zhuanlan.zhihu.com/p/268648649)

[xvfb与x11vnc_天空中的野鸟的博客-CSDN博客_xvfb](https://blog.csdn.net/qq_36383272/article/details/114970803)