---
title: Ubuntu搭建Hadoop&HBase集群
date: 2018-01-19 13:23:21
tags: ForRemember
categories:
    - Hadoop
    - HBase
---

1. 环境:
---
* 3个UbuntuServer 分别在/etc/hosts中配置ip
```
192.168.0.111 master
192.168.0.112 slave1
192.168.0.113 slave2
```
* 三台机器安装的环境
>master: jdk hadoop  hbase zookeeper  
salve1: jdk hadoop  hbase  
salve2: jdk hadoop  hbase  


2. 搭建:
---  
- ![](2.png)  
准备好这几个项目的包,去对应的官网下载即可  
    - hadoop  
    - hbase  
    - zookeeper  
    - jdk  
- 解压到你喜欢的位置,我习惯在/usr/local下安装  
配置Java,HBase,Hadoop的环境变量什么的不解释了  
![](3.png)  
环境变量,对应你的文件目录  
这里没有配置zookeeper的环境变量,也可以配置  
- zookeeper的配置文件  
![](4.png)  
启动zookeeper  
![](5.png)  
可以jps一下  
![](6.png)  
看到这个就是zookeeper的进程了

- 在$HADOOP_CONFIG_HOME是hadoop的配置文件
我们需要修改如下几个:  
1.core-site.xml  
![](7.png)  
```
<description>标签无所谓的,注意hadoop.tmp.dir指定的文件夹我们要提前建好
```
2.hadoop-env.sh
修改  
![](8.png)  
换成你的jdk位置  
![](9.png)  
3.mapred-site.xml  
![](10.png)  
4.slaves添加从节点的ip
```
slave1
slave2
```  
5.配置主从节点之间所有机器的免密ssh登陆  
    1).`ssh-keygen -t rsa`然后回车回车再回车一路确认  
    2).然后`ssh-copy-id -i /root/.ssh/id_rsa.pub master`  
    3).然后按照这个步骤将三台机器都配置好,也可以scp复制过去  
6.`cd $HBASE_HOME/conf:`  
![](12.png)    
![](13.png)  
修改上图这三项JAVA_HOME,HBASE_CLASSPATH,HBASE_MANAGES_ZK  
7.hbase-site.xml  
![](14.png)  
8.regionservers  
```
slave1
slave2
```  
9.将$HADOOP_CONFIG_HOME的core-site.xml和hdfs-site.xml复制一份到$HBASE_HOME/conf


3. 启动&验证
---
然后在master:
1.start-all.sh启动hadoop,
访问master:50070看到:  
![](16.png)  
访问master:8088:  
![](17.png)  
2.开启zookeeper  
`/usr/local/zookeeper/bin/zkServer.sh start`
3.start-hbase.sh  
访问master:16010
![](19.png)  
至此基本的的Hadoop,HBase集群就搭建完成了
