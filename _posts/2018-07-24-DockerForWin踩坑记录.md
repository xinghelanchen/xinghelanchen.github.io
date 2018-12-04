---
layout: post
title: Docker在windows上的踩坑记录
categories: Docker
description: Docker在windows上的踩坑记录
keywords: Docker, windows
---

记录一下自己的Docker学习记录。

## Docker和Idea
Docker在windows环境下的安装，这个网上一抓一大把，我这里就不赘述了，直接记录自己在使用中的一些问题吧：

### 1. 在Idea中配置docker的时候显示无法连接的问题

如下图所示：

![Image text](https://raw.githubusercontent.com/xinghelanchen/xinghelanchen.github.io/master/_img/1532441669.png)

这时候需要将图中的“tcp://192.168.99.100:2376”的tcp改为https，这样就可以连接了。

转载请标注原文链接
