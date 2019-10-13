---
title: sync_file_between_mac_linux
date: 2019-08-19 13:11:34
tags:
---

## env
### Mac
```
brew install fswatch
rsync -avz username@remotehost:/remote/path/you/want/to/sync/ ./path/you/want/to/sync/
vim watch.sh
```
```
#!/bin/zsh

fswatch /path/you/want/to/sync/ | while read file

do
rsync -rltzuq --delete  /path/you/want/to/sync/ remotehost::nnn/
echo "${file} was rsynced" >> /Users/wincher/Documents/work/leandev/rsync.log 2>&1
done
```
### nnn is defined on the remote server by rsync
#### /etc/rsync/rsync.conf
```
uid = userid
gid = groupid
use chroot = no
max connections = 10
strict modes = yes
log file = /var/logs/rsyncd/rsync.log
pid file = /var/run/rsync.pid
[nnn]
path = /path/you/want/to/sync/
comment = analyse
read only = false
hosts deny = *
hosts allow = allowed_host
```
run command below to make it work.
```
/usr/bin/rsync --daemon --config=/etc/rsync/rsync.conf
```
