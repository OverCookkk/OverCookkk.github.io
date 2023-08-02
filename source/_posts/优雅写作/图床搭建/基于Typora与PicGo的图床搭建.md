---
title: 基于Typora与PicGo的图床搭建
#tags: [hexo建站,hexo部署,github部署,个人博客]      #添加的标签
categories: 
  - 优雅写作
  - 图床搭建
description: 
#cover: 
---



## 前言

图床，就是指一些可以把图片存放到网上并且引用到其他网站使用的服务，就像以前的网络相册。本文介绍搭建的图床是基于Typora与PicGo而实现的。



## 配置Picgo

1. 在github或者gitee上（或者其他网站）建立一个仓库专门用来存放图片，github仓库一定要设置成**public**的，否则会导致图片外链不成功，另外gitee的仓库一定要**初始化**。

2. 在github中生成一个Token，用来给PicGo上传图片赋予权限。

3. 设置PicGo

   3.1 使用github

   设置github的信息，设定仓库名处要设置成用户名+仓库名。

   ![picgo](https://gitee.com/hu-zhihong/picbed/raw/master/picgo.png)

   3.2 使用gitee

   首先在PicGo中选择插件设置，安装gitee-uploader，然后repo一定要设置对应的网页链接，custompath一定要填，填default。

   ![gitee_1](https://gitee.com/hu-zhihong/picbed/raw/master/gitee_1.png)

   ![gitee_2](https://gitee.com/hu-zhihong/picbed/raw/master/gitee_2.png)

4. 设置完后，图片就可以通过PicGo上传到github的仓库里了，然后通过复制PicGo里的链接到用的地方显示图片了。

## 配置Typora

在Typora中的编好设置中设置相应的信息，然后写md文件的时候，把图片拖入到文章中，会弹出下拉框，再点击**上传图片**，则自动通过PicGo上传图片。

![Typora_1](https://gitee.com/hu-zhihong/picbed/raw/master/Typora_1.png)
