---
title: HBase Shell操作
date: 2018-01-20 15:13:20
tags:
    - HBase
    - ForRemember
categories: HBase
---

参考自:[这里](https://learnhbase.wordpress.com/2013/03/02/hbase-shell-commands/)

* Hbase环境下进入: `hbase shell`  
![pic](1.png)  
* tips: hbase shell下Backspace(退格键)失效,按住ctrl下再按就可以了切记,还有一种方法如果是XShell中可以在File->Properties->Terminal->Keyboard下，把DELETE/BACKSPACE key sequence选为ASCII 127(Ctrl+?)  

* $whoami					我是谁?  
![pic](2.png)  

# General 通用指令
* $help					    帮助
* $version					版本  
![pic](3.png)  
* $status					服务器状态  
![pic](4.png)  

# DDL操作
$create 'user','info','ext' 					创建一个表第一个为表名后面为字段(我们这是在以关系型数据库的思维来创建表,这是不适合的,我们姑且先这么创建)  
![pic](5.png)  
* $list										查看表的列表(类似于MySql的show tables;)  
![pic](6.png)  
* $describe 'user'							根据表名获取表的描述  
![pic](7.png)  
* $alter 'user','delete'=>'ext'					删除名字为ext的列族  
![pic](8.png)  
* $desc 'user'								再次查看已经成功删除掉了(desc 等同于 describe)  
![pic](9.png)  
* $is_disabled 'user'						表的状态是否为disable
* $is_enabled 'user'							表的状态是否为enable  
![pic](10.png)  
* $drop 'user'								无法删除状态为enable的表  
![pic](11.png)  
* $disable 'user'							把表设置为disable(启用为enable)  
![pic](12.png)  
* $$drop 'user'								再次drop  
![pic](13.png)  
成功删除
* $exists 'user'								查看是否存在表  
![pic](14.png)  
也刚好验证了刚才删除表的操作
再次将表建立起来
* $create 'user','info','ext'  
![pic](15.png)  
* $添加几条数据 put 'user', '你要的id名','你要输入的cf列族:列名','value'  
![pic](16.png)    
![pic](17.png)  
* $获取一个id的所有数据  
![pic](18.png)  
* $获取一个id，一个列族的所有数据  
![pic](19.png)  
* $获取一个id，一个列族中一个列的所有数据  
![pic](20.png)  
* $把id的age修改为22  
![pic](21.png)  
* $通过timestamp来获取两个版本的数据  
![pic](22.png)  
* $全表扫描  
![pic](23.png)  
* $删除键值为id的ROW(行)的'info:age'字段  
![pic](24.png)  
* $删除整行  
![pic](25.png)  
* $查询表中有多少行  
![pic](26.png)    
![pic](27.png)    
![pic](28.png)    
![pic](29.png)  
* 看到有了一行数据虽然插入两条但都是对同一个id的插入,数据在一个row上
* $为键值为'id'的row增加'info:addr'字段，并使用counter实现递增  
![pic](30.png)  
* 我们看到在第一次初始化默认值为1,再次调用自增
* 初次设置可以给初始值,然后再次调用也可以设置增加的值  
![pic](31.png)  
* 查看这个值(可以看到是按照16进制存的)  
![pic](32.png)  
* 获取当前count的值  
![pic](33.png)  
* 将整张表清空  
![pic](34.png)  

下面内容参考自->[来自](https://www.cnblogs.com/hunttown/p/5799850.html)
## 权限管理
1）分配权限
### 语法 : grant <user> <permissions> <table> <column family> <column qualifier> 参数后面用逗号分隔
### 权限用五个字母表示： "RWXCA".
### READ('R'), WRITE('W'), EXEC('X'), CREATE('C'), ADMIN('A')
### 例如，给用户‘test'分配对表t1有读写的权限，
hbase(main)> grant 'test','RW','t1'

2）查看权限
### 语法：user_permission <table>
### 例如，查看表t1的权限列表
hbase(main)> user_permission 't1'

3）收回权限
### 与分配权限类似，语法：revoke <user> <table> <column family> <column qualifier>
### 例如，收回test用户在表t1上的权限
hbase(main)> revoke 'test','t1'

###Region管理
1）移动region
### 语法：move 'encodeRegionName', 'ServerName'
### encodeRegionName指的regioName后面的编码，ServerName指的是master-status的Region Servers列表
### 示例
hbase(main)>move '4343995a58be8e5bbc739af1e91cd72d', 'db-41.xxx.xxx.org,60020,1390274516739'

2）开启/关闭region
### 语法：balance_switch true|false
hbase(main)> balance_switch

3）手动split
### 语法：split 'regionName', 'splitKey'

4）手动触发major compaction
### 语法：
```
Compact all regions in a table:
hbase> major_compact 't1'
Compact an entire region:
hbase> major_compact 'r1'
Compact a single column family within a region:
hbase> major_compact 'r1', 'c1'
Compact a single column family within a table:
hbase> major_compact 't1', 'c1'
```
## 配置管理及节点重启
1）修改hdfs配置 hdfs配置位置：/etc/hadoop/conf
## 同步hdfs配置
cat /home/hadoop/slaves|xargs -i -t scp /etc/hadoop/conf/hdfs-site.xml hadoop@{}:/etc/hadoop/conf/hdfs-site.xml
## 关闭：
cat /home/hadoop/slaves|xargs -i -t ssh hadoop@{} "sudo /home/hadoop/cdh4/hadoop-2.0.0-cdh4.2.1/sbin/hadoop-daemon.sh --config /etc/hadoop/conf stop datanode"
## 启动：
cat /home/hadoop/slaves|xargs -i -t ssh hadoop@{} "sudo /home/hadoop/cdh4/hadoop-2.0.0-cdh4.2.1/sbin/hadoop-daemon.sh --config /etc/hadoop/conf start datanode"

2）修改hbase配置
## hbase配置位置：
## 同步hbase配置
cat /home/hadoop/hbase/conf/regionservers|xargs -i -t scp /home/hadoop/hbase/conf/hbase-site.xml hadoop@{}:/home/hadoop/hbase/conf/hbase-site.xml

## graceful重启
cd ~/hbase
bin/graceful_stop.sh --restart --reload --debug inspurXXX.xxx.xxx.org
