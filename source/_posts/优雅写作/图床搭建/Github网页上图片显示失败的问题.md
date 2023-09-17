---
title: Github网页上图片显示失败的问题
#tags: [hexo建站,hexo部署,github部署,个人博客]      #添加的标签
categories: 
  - 优雅写作
  - 图床搭建
description: 
cover: https://raw.githubusercontent.com/OverCookkk/PicBed/master/blog_cover_images/391066.jpeg
---



## 问题

比如在github上打开一个项目，项目里的图片无法查看，导致在博客显示的图片也显示不了，点击**F12**打开控制台查看

![pic_fail](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/pic_fail.png)

报错显示连接超时等问题。



## 解决方法

主要思路就是使用本地`hosts`文件对网站进行域名解析，一般的`DNS`问题都可以通过修改`hosts`文件来解决，

### 找到URL

通过网页的控制台，看到图片对应的URL为`https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/git%20img1.png`

###  获取IP地址

得到上述网址以后打开[IPAddress.com](https://www.ipaddress.com/)这个网站，在搜索框输入它的域名，就是`https://`到`com`那一部分，俗称二级域名：

```text
raw.githubusercontent.com
```

在IP搜寻网站中查找这个域名对应的IP地址，如下图：

![pic_fail_2](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/pic_fail_2.png)

图片中可以看到，这个域名对应了四个IP地址。

### 修改hosts

windows系统打开：`C:\Windows\System32\drivers\etc\hosts`文件，在文件中追加

```text
185.199.108.133    raw.githubusercontent.com
185.199.109.133    raw.githubusercontent.com
185.199.110.133    raw.githubusercontent.com
185.199.111.133    raw.githubusercontent.com
```

最后在CMD中输入ipconfig/flushdns刷新DNS即可。
