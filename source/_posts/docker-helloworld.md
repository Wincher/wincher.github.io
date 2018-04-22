---
title: Docker入门(安装以及简单操作)
date: 2018-01-21 14:21:47
tags:
    - Docker
    - ForRemember
categories: Docker
---

## 环境:  
![pic](1.png)
## 安装
按理来说应该大家看这篇文章环境可能没有装过docker，但是可能有的人装过了，我们要安装的是docker-ce也就是社区版，如果你希望安装比较新的版本请看如下操作，如果你装过了，请执行接下来的操作
首先查看安装的Docker  
![pic](2.png)  
然后依次卸载掉
```
yum -y rmeove docker.x86_64
yum -y rmeove docker-client.x86_64
yum -y rmeove docker-common.x86_64
```
请注意如果没有就没必要删了，你也删不了
接下来清除余党，Docker留下来的文件  
`rm -rf /var/lib/docker/`
>然后我门添加docker-ce的源文件(不要纠结:官方源目前下载下来的版本是1.12,Docker之前是个庞大的东西，什么都做，发展到1.12版本的时候容器已经剥离出来了，在17年Docker拆分出了Moby项目，专用于容器，Docker进行产品化分为Docker CE 和Docker EE 也就是社区版和开发版本，感兴趣的话的大家可以在网上寻找Docker的发展史之类的，我们的目的就是要下载一个相对新的社区版本，应该讲明白目的了吧)

下面的命令就是为yum添加Docker CE版本的软件源  
`yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo`   
![pic](3.png)  
如果出现上图中的错误，那么就执行安装yum-config-manager  
执行:`yun -y install yum-utils`  
然后继续执行上面为yun添加Docker CE源的命令
![pic](4.png)  
然后使用yum安装docker-ce：
`yum -y install docker-ce`
安装完成后查看docker状态：
![pic](5.png)  
然后启动docker,并查看docker信息（太长就不贴图了）
```
systemctl start docker
docker info
```
![pic](6.png)  
## 简单命令
* docker ps -a查看全部容器(container)
* docker images查看镜像文件(image)，你可以理解为image是一个类class，而container就是这个类的实体，可以建立很多个实体，实体之间是隔离的
* docker search 搜索镜像
* docker pull 将镜像下载到本地
* docker docker run -dti centos：7 bash  
    -- -d代表后台运行  
    -- --name为容器添加名称
* docker exec -ti  
-i: 以交互模式运行容器，通常与 -t 同时使用  
-t: 为容器重新分配一个伪输入终端，通常与 -i 同时使用

* docker需要root权限：（生产环境风险太大）  
`usermod -aG docker yourname` 将用户添加到docker用户组，更改后要重新进入

## 扩展的问题：
* 国内访问Docker镜像库拉去镜像速度较慢,可以选择配置国内的源,比如可以使用DaoCloud提供的服务  
需要注册账号,嫌麻烦可以直接使用这个脚本,执行即可  
`curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s http://31eeb092.m.daocloud.io`
* 容器的编排（保证容器可以均匀分配资源，如何使用容器，怎么分布到各个服务器节点，容器如何通信）Mesos Kubernetes（基于docker，google开源）
* 如果系统资源不足以创建容器，会创建失败
* 容器可以独占内核，一般只会指定内存最大占比，以及最大内存值
