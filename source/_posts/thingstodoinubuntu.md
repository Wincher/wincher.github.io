---
title: 安装Ubuntu后应该做的一些事
date: 2018-01-18 11:42:11
tags: linux
categories: linux
---

1. 首先openssh是要的没有远程连接怎么能行
---  
![pic](1.png)  
查看是否是启动  
![pic](2.png)  
启动、停止和重启openssh-server的命令如下
    * /etc/init.d/ssh start
    * /etc/init.d/ssh stop
    * /etc/init.d/ssh restart

配置openssh-server
openssh-server配置文件位于/etc/ssh/sshd_config，在这里可以配置SSH的服务端口等，例如：默认端口是22，可以自定义为其他端口号，如222，然后需要重启SSH服务.
关闭ssh登录到root用户
开放ssh登录root权限是非常危险的，所以不是特别需要，应该关闭该权限，在配置文件/etc/ssh/sshd_config中找到PermitRootLogin yes一行，将yes改为no然后重启ssh即可.
Ubuntu中配置openssh-server开机自动启动
打开/etc/rc.local文件，在exit 0语句前加入：
```
/etc/init.d/ssh start
```
关于客户端连接
客户端可以用putty、SecureCRT、SSH Secure Shell Client等SSH 客户端软件，输入您服务器的IP地址，并且输入登录的用户和密码就可以登录了.我常选择的客户端软件是putty.
关于ssh的加密
实际上ssh的使用远不止这些，ssh还有很重要的一部分内容，那就是ssh通过公钥私钥进行加密，例如git就可以采用加密ssh的方式访问.关于ssh的加密这里暂不讨论，有机会再补充，可以查看相关资料了解.

2. 更换阿里源
---
```
sudo vim /etc/apt/sources.list
```
替换内容
```
# deb cdrom:[Ubuntu 16.04 LTS _Xenial Xerus_ - Release amd64 (20160420.1)]/ xenial main restricted
deb-src http://archive.ubuntu.com/ubuntu xenial main restricted #Added by software-properties
deb http://mirrors.aliyun.com/ubuntu/ xenial main restricted
deb-src http://mirrors.aliyun.com/ubuntu/ xenial main restricted multiverse universe #Added by software-properties
deb http://mirrors.aliyun.com/ubuntu/ xenial-updates main restricted
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-updates main restricted multiverse universe #Added by software-properties
deb http://mirrors.aliyun.com/ubuntu/ xenial universe
deb http://mirrors.aliyun.com/ubuntu/ xenial-updates universe
deb http://mirrors.aliyun.com/ubuntu/ xenial multiverse
deb http://mirrors.aliyun.com/ubuntu/ xenial-updates multiverse
deb http://mirrors.aliyun.com/ubuntu/ xenial-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-backports main restricted universe multiverse #Added by software-properties
deb http://archive.canonical.com/ubuntu xenial partner
deb-src http://archive.canonical.com/ubuntu xenial partner
deb http://mirrors.aliyun.com/ubuntu/ xenial-security main restricted
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-security main restricted multiverse universe #Added by software-properties
deb http://mirrors.aliyun.com/ubuntu/ xenial-security universe
deb http://mirrors.aliyun.com/ubuntu/ xenial-security multiverse
```
执行update
```
sudo apt-get update
```
![pic](3.png)  
![pic](4.png)  

3. 安装java:
---
解压`tar -zxvf`  
![pic](5.png)  
将java软连接到bin目录下  
![pic](6.png)  
配置环境变量  
![pic](7.png)  
```
export JAVA_HOME=/opt/jdk1.8.0_92
export JRE_HOME=${JAVA_HOME}/jre
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib
export PATH={JAVA_HOME}/bin:$PATH
```
![pic](8.png)  
```
source /etc/profile
```
![pic](9.png)  
![pic](10.png)  
