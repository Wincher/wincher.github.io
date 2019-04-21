---
title: Errue Occure When Apt Install
date: 2019-04-20 21:35:11
tags: ForRemember
---
### Error Occure When apt install  
```
/usr/sbin/update-info-dir: 12: /etc/default/locale: Syntax error: Unterminated quoted string
dpkg: error processing package install-info (--configure):
 installed install-info package post-installation script subprocess returned error exit status 2
Errors were encountered while processing:
 install-info
E: Sub-process /usr/bin/dpkg returned an error code (1)
```
I dont know why!  
refer to this [link](https://ubuntuforums.org/showthread.php?t=1617100):
```
Code:
sudo mv /var/lib/dpkg/info/install-info.postinst /var/lib/dpkg/info/install-info.postinst.bad
to undo,

Code:
sudo mv /var/lib/dpkg/info/install-info.postinst.bad /var/lib/dpkg/info/install-info.postinst
```
