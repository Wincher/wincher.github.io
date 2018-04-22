---
title: hadoop的一些问题
date: 2018-01-26 12:03:58
tags: ForRemember
categories: hadoop
---

- Q:Hadoop中DataNode在格式化Namenode后无法启动  
A:删除所有节点所有缓存目录，hadoop namenode -foramt
