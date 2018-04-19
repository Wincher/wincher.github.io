---
title: Hexo + GitHub Pages搭建免费博客
date: 2018-04-19 10:27:05
categories: ForRemember
tags:
    - ForRemember
    - stuff
---

## 前言
  网上有很多写的特别详尽的介绍,这篇文章主要是记录一下整个思路,操作流程,更更多的是作为备忘.

## 环境
- ### Node.js  
- ### npm
>可配置国内源 npm config set registry "https://registry.npm.taobao.org"
- GitHub账号,创建GitHub博客仓库
- 本地Git
>```
git config --global user.name "GitHub用户名"  
git config --global user.email "GitHub注册邮箱"  
#生成ssh密匙  
ssh-keygen -t rsa -C "你的GitHub注册邮箱"
cd ~/.ssh
#此时有刚刚创建的ssh密钥文件id_rsa和id_rsa.pub  
#将公钥添加到GitHub上
```

- ### Hexo
> ```
npm install hexo-cli g  
#初始化博客文件夹  
hexo init blog  
#切换到该路径  
cd blog  
#安装hexo的扩展插件  
npm install  
#安装其它可选择插件  
npm install hexo-server --save  
npm install hexo-admin --save  
npm install hexo-generator-archive --save  
npm install hexo-generator-feed --save  
npm install hexo-generator-search --save  
npm install hexo-generator-tag --save  
npm install hexo-deployer-git --save  
npm install hexo-generator-sitemap --save  
```

- ### 本地使用Hexo
> ```
#生成静态页面
hexo generate
#开启本地服务器
hexo s
#默认会提示访问本地bloghttp://localhost:4000/
#如果端口占用可设置端口启动
hexo s -p 8888
```
- ### 部署到GitHub上
> ```
#修改根目录_config.
# 参考官方文档:https://hexo.io/docs/deployment.html
deploy:
type: git
repository: git@github.com:Wincher/WincherToldYou.git
branch: master
#tips：type: git中的冒号后面由空格
#清空静态页面
hexo clean
#生成静态页面
hexo generate
#部署
hexo deploy
#也可以合成一个命令
hexo g -d
```
- ### 配置域名
> 简短说:在GitHub仓库的Setting->GitHubPages->Custom domain
配置你的域名  
在你域名的操作页面配置dns为你的GitHub Pages的公网ip地址

- ### 多终端使用
>push本地hexo中的必要文件到你的GitHub仓库中hexo分支上
```
git init
//添加需要的文件如node_modules,public就是不需要上传的文件,也可以git add .都传上去
git add source
git commit -m "init hexo source repo"
#新建hexo分支
git branch hexo
#切换到hexo分支
git checkout hexo
#本地与Github项目对接
git remote add origin git@github.com:Wincher/WincherToldYou.git
git push origin hexo
```
GitHub仓库中新建了Hexo分支，用于多终端同步博客
其他终端clone和push更新
```
#另一终端更新博客，只要将Github的hexo分支clone下来，进行初次配置
git clone -b hexo git@github.com:Wincher/WincherToldYou.git
cd  WincherToldYou.github.io
#注意，要切换到刚刚clone的文件夹内执行，安装必要的所需组件，不用再init
npm install
hexo new post "new blog name"
git add source
git commit -m "edit blog"
git push origin hexo
hexo d -g
#以后更新博客前同步下hexo的内容
```
