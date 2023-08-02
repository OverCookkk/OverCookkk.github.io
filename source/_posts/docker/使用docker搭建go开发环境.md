---
title: 使用docker搭建go开发环境
tags: [docker]      #添加的标签
categories: docker                           #添加的分类
description: 本文总结了两种docker搭建go开发环境的方法
---



## 为什么使用docker

- 可以保持系统软件环境的纯净。
- 开发环境和当前使用系统不再强依赖。无论系统再怎么升级，都不会再影响到我的开发环境。
- 开发软件的管理方式更加统一。各种编程语言都有各自的安装流程和步骤，各种应用服务的安装和配置方式也千差万别。通过 Docker，从一个更高的维度抽象和统一了这些差异。不论是 MySQL，还是 Redis，我都只需要拉镜像，映射端口，然后启动容器就行了。



## 搭建方法一

### 测试镜像

使用docker来构建开发环境的第一步就是先找镜像。在 docker hub 上就有 Go 语言的官方镜像：**golang**，先用下面的命令测试一下这个镜像。

`docker run --rm golang:alpine go version`

- 这条命令以 `go` 为分界线，前面的部分属于 docker，表示执行 **golang** 镜像容器。 `alpine` 是这个镜像的标签，我个人喜欢 **alpine** 的小巧，所以选择的这个。
- `--rm` 参数表示执行完成后就删除容器，避免浪费存储空间。



### 执行go代码

上面的 go 命令虽然可以执行了，不过还不能运行 go 代码文件。因为还没提供目录挂载功能。自定义的 Shell 脚本内容如下：

```shell
#!/bin/bash

docker run --rm \
    -v $PWD:/srv/app \
    -w /srv/app \
    golang:alpine myTest
```
- myTest是编译好的go程序。
- `-v` 将宿主主机目录挂在到容器里。`$PWD` 代表当前执行命令的位置，即宿主机目录，`/srv/app` 是镜像容器中的目录。
- `-w` Docker 设置容器运行时的工作主目录。这主要是为了配合上面的 `-v` 参数来执行当前目录下的 go 代码

注释：**将宿主机目录挂载到容器里 **   这句话可以理解成   把容器想成一个单独的系统，或者说电脑，而你的宿主机目录是一个U盘，挂载后，你往宿主机该目录里放文件，那么通过容器里对应目录便可以访问到此文件，不需要重新生成容器就可以在“容器外部”添加和修改某些文件。



### 修改go环境变量

Go 可以通过环境变量的方式来配置镜像，Docker 也支持设置容器运行时的环境变量。

```shell
#!/bin/bash

docker run --rm \
	-e GOPROXY=https://goproxy.cn \
    -v $PWD:/srv/app \
    -w /srv/app \
    golang:alpine myTest
```

添加了一个 `-e` 参数，这是 Docker 用来设置容器运行时的环境变量，通过这个参数把后面 Go 的镜像家属配置带入运行的容器。





## 搭建方法二

编写Dockerfile文件，通过文件来创建镜像

```dockerfile
# 使用基础镜像来创建我们的镜像，后续的指令都基于该基础镜像环境运行
FROM golang:alpine

# 为我们的镜像设置必要的环境变量
ENV GO111MODULE=on \
    CGO_ENABLED=0 \
    GOOS=linux \
    GOARCH=amd64

# 移动到容器中的工作目录：/build
WORKDIR /build

# 将本地代码复制到容器中
COPY . .

# 将我们的代码编译成二进制可执行文件http_mytest
# 下载gin依赖包
RUN go mod init build
RUN go get -u github.com/gin-gonic/gin
RUN go build -o http_mytest .

# 移动到容器中用于存放生成的二进制文件的 /dist 目录
WORKDIR /dist

# 将二进制文件从 /build 目录复制到这里
RUN cp /build/http_mytest .

# 声明服务端口
EXPOSE 8888

# 运行镜像（启动容器）时运行的命令
CMD ["/dist/http_mytest"]
```

- FROM:文件的开始

  FROM centos	#使用centos为系统，若没有则拉取

- LABEL：相当于注释或者说明信息

  LABEL version="1.0"

  LABEL author="xxx"

- RUN:执行命令，每执行一条RUN，就会多一层，**所有RUN使用&&连接多个命令一起执行**

  RUN yum -y update

- WORKDIR:在容器中进入或创建目录

  WORKDIR /test

- ADD：将本地文件添加到镜像里

  ADD可以解压缩文件

  ADD  hello /	# 把hello添加到镜像里的根目录

  ADD xxx.tar.gz /	#添加并解压到根目录

- COPY：将本地文件添加到镜像里

  WORKDIR /root/test

  COPY hello .	# /root/test/hello

- ENV：

  ENV MYSQL_VERSION 5.6	# 设置常量

  RUN apt-get -y install mysql-server="{MYSQL_VERSION}"

- CMD and ENTRYPOINT

  CMD ["python", "app.py"]

  若docker指定了其他命令，CMD会被忽略
  
  若定义多个CMD，只会执行最后一个CMD



### 构建镜像

在项目目录下，执行下面的命令创建镜像，并指定镜像名称为`goweb_app`：

```bash
docker build -t rerank_service:v0.0.1 .
```

- -t：给镜像加一个Tag
- `rerank_service`是镜像名
- `v0.0.1`是tag名
- `.` 表示当前目录，即Dockerfile所在目录

现在我们已经准备好了镜像，但是目前它什么也没做。我们接下来要做的是运行我们的镜像，以便它能够处理我们的请求。运行中的镜像称为容器。



### 运行镜像

执行下面的命令来运行镜像：

```bash
docker run --rm -d -p 8888:8888 goweb_http
```

`-d`是让该容器运行在后台。

标志位`-p`用来定义端口绑定。由于容器中的应用程序在端口8888上运行，我们将其绑定到主机端口也是8888。如果要绑定到另一个端口，则可以使用`-p $HOST_PORT:8888`。例如`-p 5000:8888`。



### 进入容器

执行下面命令来进入到Dockerfile中拉取的基础镜像环境中：

```bash
docker exec -it 容器id /bin/sh
```

```bash
/dist #ls
http_mytest
```



### 查看容器日志

```bash
docker logs -f 容器id
```

-f：实时打印日志



```bash
docker logs --tail 50 容器id
```



