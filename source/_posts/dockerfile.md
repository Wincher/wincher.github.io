---
title: DOCKERFILE构建image
date: 2018-01-22 14:45:46
tags: Docker
categories: Docker
---

## 演示一个最基本的dockerfile构建image
使用dokerfile制作一个带java环境的centos  
首先准备好随意一个目录,里面需要有一个命名为Dockerfile的文件,并将jdk放到这个文件夹下  
![pic](1.png)  
我们来看Dockerfile的内容  
![pic](2.png)  
ADD指令可以解压tar指令能够解压的压缩包,CP命令拷贝压缩包，或者文件夹最后不带/也是直接传入压缩包  
然后在这个文件夹内执行:
`docker build -t centos7_java:1.8 .`  
-t 指定tag  
.指定执行的Dockerfile所在的目录  
![pic](3.png)  
然后执行`docker images` 查看docker的镜像  
![pic](2.png)  
我们发现多了一个名称为centos7_java,tag为1.8的镜像
