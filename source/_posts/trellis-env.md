---
title: trellis_env
date: 2019-04-20 11:22:20
tags:
---

### env  
Ubuntu18.04  
### dependencies
* PHP  
`sudo apt install php`  
* composer  
```
wget https://getcomposer.org/composer.phar
sudo mv composer.phar /usr/local/bin/composer
chmod +x /usr/local/bin/composer
```
* Vagrant  
`sudo apt install vagrant`  
* VirtualBox  
[refer to](https://www.virtualbox.org/wiki/Linux_Downloads)  

### build trellis project  
```
mkdir example.com && cd example.com
git clone --depth=1 https://github.com/roots/trellis.git && rm -rf trellis/.git
composer create-project roots/bedrock site
```
### More Detail
[RTFM](https://roots.io/trellis/docs/)
