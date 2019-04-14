---
title: nginx_guide
date: 2019-04-14 15:14:39
tags: ForRemember
---

### Download Nginx  
find link in [Nginx_download](http://nginx.org/en/download.html)
`wget http://nginx.org/download/nginx-1.14.2.tar.gz`  
`tar -zxvf nginx nginx-1.14.2.tar.gz -C <path>`  
`cd path/nginx-1.14.2 && ll`  
![pic](1.png)  
### Directory Structure  
1. auto  
support configure script
`ll auto`  
![pic](2.png)  
2. conf  
template of nginx conf  
3. contrib  
![pic](3.png)  
let your vim know nginx syntax
`cp -r contrib/vim/* ~/.vim`  
5. man  
`man man/nginx.8`  
6. html  
50x and index default html file for nginx  
7. src  
![pic](4.png)  
### Compile  
check attribute of configure  
`./configure --help`  
**--with-** module will not compile by default  
**--without-** module will compile by default  
better use **absolute path**
`./configure --prefix=/prefix_path`  
an objs dir is new during ./configure running
![success_pic](5.png)  
`make`  
**nginx** binary file is generate in ./objs, all dynamic module are here  
`make install`  
`cd prefix_path`  
`sudo ./sbin/nginx`
now by default, you can visit nginx by server's 80 port  

###

### Hot-Deploy  
`cp nginx nginx.old`
`cp -f somepath/nginx nginx`
`kill -USR2 pid_of_old_nginx`
a new nginx process will start..  
tell old version nginx shutdown gracefully, master process of old nginx will not stop, because we could cancel update nginx  
`kill -WINCH pid_of_old_nginx`
