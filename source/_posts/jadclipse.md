---
title: jadclipse
date: 2018-01-20 16:19:41
tags: ForRemember
categories: tools
---
- [下载JadClipse](http://jadclipse.sourceforge.net/wiki/index.php/Main_Page#Download)，注意选择与eclipse版本一致的版本，我用的是Eclipse最新版，所以选择下载最新版net.sf.jadclipse_3.3.0.jar  
- [下载Jad](http://www.varaneckas.com/jad)，下载相应版本jad158g.win.zip，下载后解压  
![](1.png)  
- 解压jad158g.win.zip得到文件  
![](2.png)  
- 将Jad.exe拷贝到JDK安装目录下的bin文件下（方便，与java，javac等常用命令放在一起，可以直接在控制台使用jad命令），我的机器上的目录是D:\environment\Java\jdk1.8.0_152\bin  
将下载下来的Jadclipse，如net.sf.jadclipse_3.3.0.jar拷贝到Eclipse下的plugins目录即可。当然也可以用links安装.
- 然后，重新启动Eclipse，找到Eclipse->Window->Preferences->Java，此时你会发现会比原来多了一个JadClipse的选项，单击，会出现，如下：
在Path to decompiler中输入你刚才放置jad.exe的位置，也可以制定临时文件的目录，如图所示  
![](3.png)  
- 基本配置完毕后，我们可以查看一下class文件的默认打开方式，Eclipse->Window->Preferences->General->Editors->File Associations，我们可以看到下图：  
![](4.png)  
- 按步骤顺序操作,第3步add在4输入jad(由于我已经添加,所以显示空白),然后执行5,6设置为default,以后访问没有源码的class文件就自动进入反编译的页面了
