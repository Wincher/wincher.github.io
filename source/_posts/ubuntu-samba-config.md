---
title: ubuntu_samba_config
date: 2020-07-29 11:10:10
tags:
---

# target
share files on intranet

# precondition
* a running ubuntu server
* a folder you wanna shared

# step
```
apt install samba
vi /etc/samba/smb.conf
```
configuration add below
```
[share]
    path=<folder_shared>
    available=yes
    browseable=yes
    public=yes
    writable=yes
```

if you need password
```
touch /etc/samba/smbpasswd
smbpasswd - a test
```
then restart...
`systemctl restart smdb`
