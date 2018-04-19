---
title: Hive安装
date: 2018-02-06 16:57:00
categories: Hive
tags: Hive
---

## 目标
安装一个基本的Hive环境,由于基本配置的安装,暂时先使用Hive自带的Derby作为元数据存储数据库,当然完全体还是要使用Mysql之类的数据库

### 环境:
Ubuntu 16.04 Server  
已部署Hadoop2.8.3并启动
[详情](http://wincher.cn)

### 下载Apache Hive
>[下载地址](http://hive.apache.org/downloads.html)
我选择了下载2.3.2版本

### 安装Apache Hive
1. 解压缩Hive  
    `sudo tar -zxvf apache-hive-2.3.2-bin.tar.gz`
1. 讲解压的apache-hive-2.3.2-bin文件夹移动到/usr/local下并更名为hive</br>
    `mv apache-hive-2.3.2-bin /usr/local/hive`
1. 解压Hive  
    `sudo tar -zxvf apache-hive-2.3.2-bin.tar.gz`
1. 配置环境变量  
    `vim /etc/profile`  
    在文件尾部添加  
    ```
    export HIVE_HOME=/usr/local/hive
    export PATH=$PATH:$HIVE_HOME/bin
    ```
    执行source命令是配置生效  
    `source /etc/profile`

### Hive配置Hadoop HDFS
1. 进入$HIVE_HOME/conf,复制hive-env.sh.template为hive-env.sh
    `cd $HIVE_HOME/conf`
    `cp hive-env.sh.template hive-env.sh`
1. 编辑hive-site.xml配置文件  
    `vim hive-env.sh`  
    添加如下内容
    ```
    export HADOOP_HOME=/usr/local/hadoop <!-- 你的Hadoop的位置 -->
    export HIVE_CONF_DIR=/usr/local/hive/conf
    export HIVE_AUX_JARS_PATH=/usr/local/hive/lib
    ```
1. 进入$HIVE_HOME/conf,复制hive-default.xml.template为hive-site.xml
    ```
    cd $HIVE_HOME/conf  
    cp hive-default.xml.template hive-site.xml
    ```
1. 编辑hive-site.xml配置文件,替换所有的{$system:java.io.tmpdir}为/usr/local/hive/tmp  
    替换所有的{$system:user.name}为root
    ```
    vim hive-site.xml
    :%s/{$system:java.io.tmpdir}/\/user\/local\/hive\/tmp/g
    :%s/{$system:user.name}/root/g
    :wq
    ```
    并修改如下几个配置：  
    metastore路径
    ```
    <property>
        <name>javax.jdo.option.ConnectionURL</name>
        <value>jdbc:derby:;databaseName=/usr/local/hive/bin/metastore_db;create=true</value>
        <description>
          JDBC connect string for a JDBC metastore.
          To use SSL to encrypt/authenticate the connection, provide database-specific SSL flag in the connection URL.
          For example, jdbc:postgresql://myhost/db?ssl=true for postgres database.
        </description>
      </property>
    ```  
    hdfs中的路径(这个我们后面要在Hdfs中创建)
    ```
    <property>
        <name>hive.exec.scratchdir</name>
        <value>/tmp/hive</value>
        <description>HDFS root scratch dir for Hive jobs which gets created with write all (733) permission. For each connecting user, an HDFS scratch dir: ${hive.exec.scratchdir}/&lt;username&gt; is created, with ${hive.scratch.dir.permission}.
        </description>
    </property>
    ```  
    本地路径
    ```
    <property>
        <name>hive.exec.local.scratchdir</name>
        <value>/usr/local/hive/tmp/root</value>
        <description>Local scratch space for Hive jobs</description>
    </property>
    ```

1. 使用Hadoop新建hdfs目录,对用hive-site.xml中有如下配置：
    ```
    <property>
        <name>hive.metastore.warehouse.dir</name>
        <value>/user/hive/warehouse</value>
        <description>location of default database for the warehouse</description>
        </property>
    <property>
    ```
    执行hadoop命令新建/user/hive/warehouse目录  
    给新建的目录赋予读写权限  
    查看修改后的权限  
    ```  
    hadoop dfs -mkdir -p /user/hive/warehouse
    hdfs dfs -chmod 777 /user/hive/warehouse
    hdfs dfs -ls /user/hive
    Found 1 items
    drwxrwxrwx   - root supergroup          0 2018-02-06 16:05 /user/hive/warehouse
    ```  
    运用hadoop命令新建/tmp/hive目录(这个对应上面的配置hive.exec.scratchdir)  
    给目录/tmp/hive赋予读写权限  
    检查创建好的目录  
    ```
    hdfs dfs -mkdir -p /tmp/hive  
    hdfs dfs -chmod 777 /tmp/hive
    hdfs dfs -ls /tmp
    Found 1 items
    drwxrwxrwx   - root supergroup          0 2018-02-06 17:04 /tmp/hive
    ```

### 使用HIVE
1. 初始化derby
    schematool -initSchema -dbType derby
1. 使用(注意使用前需要启动Hadoop的Hdfs和Yarn，start-all.sh包含了这两个的启动)
    命令行输入,会出现HQL命令行提示符(会有几行log)
    ```
    root@master:/usr/local/hive/conf# hive
    hive>
    ```
    这样就安装完成了，接下来是一些简单的操作:
    1. 查看函数列表  
    `hive>show functions;`
    1. 查看某个函数的详细信息如:sum
    ```
    hive> desc function sum;
    OK
    sum(x) - Returns the sum of a set of numbers
    Time taken: 0.013 seconds, Fetched: 1 row(s)
    ```
    1. 新建库并使用  
    ```
    hive> create database test;
    OK
    Time taken: 0.448 seconds
    ```
    1. 新建数据表并查看详情  
    ```
    hive> create table person(id int, name string) row format delimited fields terminated by '\t';
    OK
    Time taken: 0.16 seconds
    hive> desc person;
    OK
    id                  	int
    name                	string
    Time taken: 0.083 seconds, Fetched: 2 row(s)
    ```
    1. 将文件写入表中  
    在$HIVE_HOME下新建文件person.date,并在文件中添加如下内容,  
    注意id和name间是TAB键,我们在建表语句中用了terminated by '\t',所以这个分割是必须要用TAB键的

    ```
    001    Tiny
    002    Lina
    003    Zues
    004    Sven
    005    Riki
    006    Park
    007    Morphling
    008    Riki
    009    Morphling
    010    Morphling
    ```
    1. 将数据倒入并查看是否成功
    ```
    hive> load data local inpath '/usr/local/hive/person.dat' into table test.person;
    Loading data to table test.person
    OK
    Time taken: 9.767 seconds
    hive> select * from test.person;
    OK
    1	Tiny
    2	Lina
    3	Zues
    4	Sven
    5	Riki
    6	Park
    7	Morphling
    8	Riki
    9	Morphling
    10	Morphling
    Time taken: 2.671 seconds, Fetched: 10 row(s)
    ```
    1. 在Hadoop的NameNode上也能查看到刚写入HDFS的数据person.dat  
    http://master:50070/explorer.html#/user/hive/warehouse/test.db/person
    ![pic](ScreenShot20180206at163905.png)
