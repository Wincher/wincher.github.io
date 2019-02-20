---
title: ansible初识
date: 2019-02-17 00:19:10
tags:
    --ForRemember
categories:
    --operation
---

## Why
Every time when install a new server, it really take some time to install these or those tools, i want to make this shit a auto way, and Ansible is born for that.

## Prepare
What you should need to do for the server is just to install it, for me, ubuntu 18.04, and just make sure ssh works,  and then you can leave that server alone.

## install ansible
pip install ansible

##
touch /etc/ansible/hosts
vim /etc/ansible/hosts
```
[test]  # group name
192.168.1.103   #remote server address
```
ansible test -m ping
```
192.168.1.103 | FAILED! => {
    "changed": false,
    "module_stderr": "Shared connection to 192.168.1.103 closed.\r\n",
    "module_stdout": "/bin/sh: 1: /usr/bin/python: not found\r\n",
    "msg": "MODULE FAILURE\nSee stdout/stderr for the exact error",
    "rc": 127
}
```
that's because it cant find python in path:/usr/bin/python on remote server, for me ubuntu 18.04 only installed python3, there are 2 solution.
1)`sudo apt install python` on remote server.
2) edit /etc/ansible/hosts to config ansible_python_interpreter:
eg:192.168.1.103 ansible_python_interpreter=/usr/bin/python3
or maybe) you can link python3 to python on remote server (not recommended)
```
192.168.1.103 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```  
