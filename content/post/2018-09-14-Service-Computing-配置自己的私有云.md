---
title: "Service Computing 实验一配置自己的私有云"
date: 2018-09-14T11:38:52+08:00
lastmod: 2018-09-14T11:41:52+08:00
menu: "main"
weight: 50
author: "hansenbeast"
tags: [
    "Service Computing"
]
categories: [
    "Tech Blogs"
]
# you can close something for this content if you open it in config.toml.
comment: false
mathjax: false
---



# Sevice Computing

**Principle, Technology and Architecture for building effitive, elastic and solid services on cloud**

## 概述：

上学期有幸学习了云计算概论这门课，初窥了一直很热门的却又陌生的云计算，以我个人粗浅的理解，它是依赖于由WSC（仓储级计算机）构成的巨型数据中心，即性能强大的底层基础设施，通过虚拟化的技术和互联网，提供PaaS（软件即服务）的其中一种交付方式，而软件开发者可以利用云平台开发运算能力更强大的应用程序，尤其是针对大数据的处理和分析，为用户减少了时间和空间上的消耗。按需使用和弹性的服务技术也为企业减少了不少维护成本。

![1](Assets/1.png)

## 实验1 配置自己的私有云，使其提供远程云桌面服务

实验环境

- macos  high sierra 10.13.4
- VMware Fusion

> **虚拟化是一种将功能与硬件分离的技术，而云则建立在这种分离技术之上。由于两者的核心理念都是从抽象资源中创建可用的环境，所以很容易被混为一谈。** **虚拟化的主要功能是将单个资源抽象为多个提供给用户使用，而云计算则帮助不同部门（通过私有云）或公司（通过公共云）访问一个自动配置的资源池。**

### 1、使用VMware Fusion安装虚拟机镜像

什么是虚拟机：

> 虚拟计算机系统被称为“虚拟机”(VM)，它是一种严密隔离的软件容器，内含操作系统和应用。每个功能完备的虚拟机都是完全独立的。通过将多台虚拟机放置在一台计算机上，可仅在一台物理服务器或“主机”上运行多个操作系统和应用。名为“hypervisor”的精简软件层可将虚拟机与主机分离开来，根据需要为每个虚拟机动态分配计算资源。

 首先通过教育网连接清华开源镜像站高速下载[centos7-Minimal-ISO](https://mirror.tuna.tsinghua.edu.cn/centos/7.5.1804/isos/x86_64/)，安装过程没有遇到什么问题就不赘述了。

![2](Assets/2.png)



因为之后需要用到GNOME桌面环境和Chrome浏览器，所以虚拟机配置采用2核，2G内存，30G磁盘。

![20](Assets/20.png)



### 2、不得不说的网络设置

VMWare为我们提供了三种网络模式，分别是bridged(桥接模式)、NAT(网络地址转换模式)和host-only(仅主机模式)。

![21](Assets/21.png)

1. 桥接网络：

   > 是指本地物理网卡和虚拟网卡通过VMnet0虚拟交换机进行桥接，物理网卡和虚拟网卡由于在拓扑图上处于同等地位，所以就相当于处于同一个网段，**虚拟交换机就相当于一台现实网络中的交换机,所以两个网卡的IP地址不同，但要设置为同一网段。**

![22](Assets/22.png)

比如宿舍里有一个路由器，宿舍里四个人连接这个路由器，默认是192.168.1.1,子网掩码是255.255.255.0。而其他四个人是自动获取IP（DHCP），假设四个人的IP是**192.168.1.2到192.168.1.5**。那么虚拟机可以设置的IP地址是**192.168.1.6到192.168.1.254**(网络地址全0和全1的除外，再除去四个人的IP地址)，这样虚拟机就相当于宿舍里的一个新同学一样，多分配了一个地址。如果虚拟机需要上外网，那么还需要配置虚拟机的路由地址，就是192.168.1.1了，这样，虚拟机就可以上外网了，但是，上网我们一般是通过域名去访问外网的，所以我们还需要为虚拟机配置一个dns服务器，我们可以简单点，把dns服务器地址配置为google的dns服务器:8.8.8.8,到此，虚拟机就可以上网了。 所以当我们要在局域网使用虚拟机，对局域网其他PC提供服务时，例如提供ftp，ssh，http服务，那么就要选择桥接模式。

![23](Assets/23.png)  

2. NAT：

> 虚拟机的网卡连接到宿主的 VMnet8 上。此时系统的 VMWare NAT Service 服务就充当了路由器的作用，负责将虚拟机发到 VMnet8 的包进行地址转换之后发到实际的网络上，再将实际网络上返回的包进行地址转换后通过 VMnet8 发送给虚拟机。VMWare DHCP Service 负责为虚拟机提供 DHCP 服务。

  

![24](Assets/24.png)![25](Assets/25.png)

NAT和桥接的比较:

1. NAT模式和桥接模式虚拟机都可以上外网。
2. 由于NAT的网络在vmware提供的一个虚拟网络里，所以局域网其他主机是无法访问虚拟机的，而宿主机可以访问虚拟机，虚拟机可以访问局域网的所有主机，因为真实的局域网相对于NAT的虚拟网络，就是NAT的虚拟网络的外网。
3. 桥接模式下和NAT模式下，多个虚拟机之间都可以互相访问。

**如上上图中，A1，A2可以访问B，但B不可以访问A1，A2。但A，A1，A2可以互访。**

使用场景：   

比如建多个虚拟机集群，在虚拟机中不用进行任何手工配置就能直接访问互联网，建议你采用NAT模式。



3.Host-Only：

> 在Host-Only模式下，虚拟网络是一个全封闭的网络，它唯一能够访问的就是宿主机。其实Host-Only网络和NAT网络很相似，不同的地方就是Host-Only网络没有外网服务，不能连接到Internet。主机和虚拟机之间的通信是通过VMware Network Adepter VMnet1虚拟网卡来实现的。

![26](Assets/26.png)

![27](Assets/27.png)

**上图中A，A1，A2可以互访，但A1，A2不能访问B，也不能被B访问。**

Host-Only的宗旨就是建立一个与外界隔绝的内部网络，来提高内网的安全性。这个功能或许对普通用户来说没有多大意义，但大型服务商会常常利用这个功能。

> 参考博客：
>
> https://blog.csdn.net/CleverCode/article/details/45934233
>
> https://www.cnblogs.com/ggjucheng/archive/2012/08/19/2646007.html



本次实验需要为虚拟机机创建了两张虚拟网卡，分别是

1. NAT模式，如上所述它会自动地通过宿主机上的 VMnet8虚拟交换机来共享主机的网络连接。
2. 仅主机模式，它会连接宿主机上的 VMnet1虚拟交换机来接入到这个局域网中。故在虚拟机中设置两个网络适配器，一个连外网，一个供宿主机和虚拟机之间进行专用连接。（关机状态下设置）。

![6](Assets/6.png)

首先在宿主机通过ifconfig命令查看Host-Only模式那块虚拟网卡的地址（即vmnet1）和NAT模式虚拟网卡的地址（即vmnet8）。（其中vmnet2是自定义的一张虚拟网卡，没有用到）

![28](Assets/28.png)

![29](Assets/29.png)

然后启动配置好的虚拟机后，还需要通过nmtui命令启动两块网卡的服务。其中ens33为NAT网络服务，ens37为专用网络网络服务。

![5](Assets/5.png)

需要再通过ip addr命令查看虚拟机在专用网络中的IP地址，以及通过NAT获得的IP地址。以便主机ping和ssh。发现内网IP地址为172.16.162.128，外网IP为192.168.218.131，且前24位均与宿主机中的相同，说明和宿主机处于同一个子网中。

![30](Assets/30.png)

虚拟机访问外网成功。

![31](Assets/31.png)

虚拟机使用内网和外网IP均可以访问ping宿主机。

![32](Assets/32.png)

最后宿主机ssh虚拟机成功。

![33](Assets/33.png)

![8](Assets/8.png)

### 3、升级OS内核

CentOS7默认有浙江大学的镜像源，能够利用高速的教育网来完成 yum 的包安装和更新。如果遇到问题请[更换yum源](https://www.cnblogs.com/wujinghua/p/8552785.html)

```bash
yum update
yum install wget
yum install kernel-devel  #安装内核的头文件，开发需要用到
```

![10](Assets/10.png)



### 4、拷贝多个虚拟机

为了提高云服务的可靠性和弹性，需要在一台宿主机上配置多个冗余的虚拟机以应对出现问题时的数据迁移。在 VMWare中，包括许多VPS的云托管商，首先会创建一个当前虚拟机的快照，然后用快照生成一个与原虚拟机独立的系统。

其中有两种不同的复制方式：**链接复制** 和 **完整复制**。其中，链接复制的虚拟机虽然和原虚拟机在运行上是独立的，但是仍旧使用原系统的磁盘空间，而所有对于原虚拟机快照中文件的修改将会被存储在一个“差量磁盘”上。这种方法的好处是省去了创建新虚拟磁盘的时间，使得两个虚拟机共享一套快照文件，节约磁盘空间。而后者则是单独开辟虚拟磁盘，并在之上创建虚拟机。

![11](Assets/11.png)

需要注意的是，虚拟机中网卡的 MAC 地址必须不同，同样使用nmtui开启两块网卡，因为宿主机的虚拟网卡（Host-Only）均配置了 DHCP 服务，子网中的设备会自动获得不冲突的 IP 地址。

![34](Assets/34.png)

宿主机ssh克隆机成功。

![35](Assets/35.png)

![13](Assets/13.png)



### 5、配置桌面环境

```bash
#安装桌面 
yum groupinstall "GNOME Desktop" 
#设置图形化组建为启动目标
ln -sf /lib/systemd/system/runlevel5.target /etc/systemd/system/default.target 
#重启
shutdown -r now 
```

启动两个网络服务。

![14](Assets/14.png)



[CentOS7安装chrome浏览器](https://www.itzgeek.com/how-tos/linux/centos-how-tos/install-google-chrome-37-on-centos-7-rhel-7.html)

[CentOS7安装vscode](https://code.visualstudio.com/docs/setup/linux#_rhel-fedora-and-centos-based-distributions)

### 6、远程桌面连接

因为苹果上的apple remote收费，所以安装MS为macOS用户提供的[Mircosoft Remote Desktop （远程桌面连接）](https://zhuanlan.zhihu.com/p/34202380)

添加虚拟机IP。（我分别添加了内网和外网的IP）

![36](Assets/36.png)

![19](Assets/19.png)

因为微软的远程桌面连接软件支持的是`rdp` 协议，而 Linux 中需要通过额外的组件 `Xrdp` 来支持这种协议的远程桌面。所以在连接虚拟机之前，需要配置安装这个组件。

首先配置 EPEL 源，这是对于 CentOS 原生 yum 源的补充。

```bash
#sudo获得管理员权限
yum install -y epel-release.noarch #如果没有可用yum search epel搜索相关源 
yum install xrdp 
yum install tigervnc-server
```

启动xrdp

```bash
systemctl start xrdp
systemctl enable xrdp #添加开机启动
```

查看端口

```bash
netstat -antup | grep xrdp
```

![15](Assets/15.png)

**此时3389端口负责监听rdp协议。**

然后关闭防火墙，将端口号开放，同时为 XRDP 开启 Selinux (控制程序的访问权限)认证

```bash
firewall-cmd --permanent --add-port=3389/tcp
firewall-cmd --reload
chcon --type=bin_t /usr/sbin/xrdp
chcon --type=bin_t /usr/sbin/xrdp-sesman
```

最后尝试连接。

内网IP。

![37](Assets/37.png)

外网IP

![16](Assets/16.png)

![18](Assets/18.png)

大功告成。

![17](Assets/17.png)