---
title: nginx
#tags: [负载均衡]      #添加的标签
categories: 
  - 中间件
description: 
cover: https://raw.githubusercontent.com/OverCookkk/PicBed/master/blog_cover_images/00730-647474016.png
---



## 简介

  **Nginx** 是开源的轻量级 Web 服务器、反向代理服务器，以及负载均衡器和 HTTP 缓存器。其特点是高并发，高性能和低内存。
		**Nginx** 专为性能优化而开发，性能是其最重要的考量，实现上非常注重效率，能经受高负载的考验，最大能支持 50000 个并发连接数。 Nginx 还支持热部署，它的使用特别容易，几乎可以做到 7x24 小时不间断运行。 Nginx 的网站用户有：百度、淘宝、京东、腾讯、新浪、网易等。





## 代理

### 正向代理

​		**Nginx** 不仅可以做反向代理，实现负载均衡，还能用做正向代理来进行上网等功能。

![nginx正向代理示意图](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/nginx%E6%AD%A3%E5%90%91%E4%BB%A3%E7%90%86%E7%A4%BA%E6%84%8F%E5%9B%BE.png)





### 反向代理

​		客户端对代理服务器是无感知的，客户端不需要做任何配置，用户只请求反向代理服务器，反向代理服务器选择目标服务器，获取数据后再返回给客户端。反向代理服务器和目标服务器对外而言就是一个服务器，只是暴露的是代理服务器地址，而隐藏了真实服务器的IP地址。

![nginx反向代理示意图](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/nginx%E5%8F%8D%E5%90%91%E4%BB%A3%E7%90%86%E7%A4%BA%E6%84%8F%E5%9B%BE.png)





## 负载均衡

​		将原先请求集中到单个服务器上的情况改为增加服务器的数量，然后将请求分发到各个服务器上，将负载分发到不同的服务器，即负载均衡。





## 动静分离

 	为了加快网站的解析速度，可以把静态页面和动态页面由不同的服务器来解析，加快解析速度，降低原来单个服务器的压力。

![nginx动静分离示意图](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/nginx%E5%8A%A8%E9%9D%99%E5%88%86%E7%A6%BB%E7%A4%BA%E6%84%8F%E5%9B%BE.png)
