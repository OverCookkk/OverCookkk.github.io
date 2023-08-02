---
title: k8s简介
tags: [k8s]      #添加的标签
categories: k8s		#添加的分类
description: 
---

## 容器编排

容器化部署方式给带来很多的便利，但是也会出现一些问题，比如说：

- 一个容器故障停机了，怎么样让另外一个容器立刻启动去替补停机的容器。

- 当并发访问量变大的时候，怎么样做到横向扩展容器数量。

这些容器管理的问题统称为容器编排问题。



## 编排系统的需求催生 Kubernetes 

尽管Docker为容器化的应用程序提供了开放标准，但随着容器越来越多出现了一系列新问题：

- 如何协调和调度这些容器？
- 如何在升级应用程序时不会中断服务？
- 如何监视应用程序的运行状况？
- 如何批量重新启动容器里的程序？

解决这些问题需要容器编排技术，可以将众多机器抽象，对外呈现出一台超大机器。现在业界比较流行的有：Kubernetes 、Mesos、Docker Swarm。

在业务发展初期只有几个微服务，这时用 Docker 就足够了，但随着业务规模逐渐扩大，容器越来越多，运维人员的工作越来越复杂，这个时候就需要编排系统解救。



## Kubernetes 解决的核心问题

- 自我修复：一旦某一个容器崩溃，能够在1秒中左右迅速启动新的容器。
- 弹性伸缩：可以根据需要，自动对集群中正在运行的容器数量进行调整。
- 服务发现：服务可以通过自动发现的形式找到它所依赖的服务。
- 负载均衡：如果一个服务起动了多个容器，能够自动实现请求的负载均衡。

- 版本回退：如果发现新发布的程序版本有问题，可以立即回退到原来的版本。

- 存储编排：可以根据容器自身的需求自动创建存储卷。

Kubernetes 的出现不仅主宰了容器编排的市场，更改变了过去的运维方式，不仅将开发与运维之间边界变得更加模糊，而且让 DevOps 这一角色变得更加清晰，每一个软件工程师都可以通过 [Kubernetes](https://link.juejin.cn?target=http%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzI0MDQ4MTM5NQ%3D%3D%26mid%3D2247510899%26idx%3D1%26sn%3D6c136480f2812261681309aeac17d73e%26chksm%3De918ce6fde6f4779a86d1f78fd123310614a1ae895e8a3bece27c68f7f434e86a4a65746637e%26scene%3D21%23wechat_redirect) 来定义服务之间的拓扑关系、线上的节点个数、资源使用量并且能够快速实现水平扩容、蓝绿部署等在过去复杂的运维操作。



## Kubernetes  架构和组件

Kubernetes 由众多组件组成，组件间通过 API 互相通信，归纳起来主要分为三个部分：

- controller manager
- nodes
- pods

![k8s集群架构图](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/k8s%E9%9B%86%E7%BE%A4%E6%9E%B6%E6%9E%84%E5%9B%BE.png)

- **ApiServer** : 资源操作的唯一入口，接收用户输入的命令，提供认证、授权、API注册和发现等机制。
- **Scheduler** : 负责集群资源调度，按照预定的调度策略将Pod调度到相应的node节点上。

- **Controller Manager**：即控制管理器集合，这些控制器都具有一种名为控制循环（control loop）的机制，这个机制的功能简单来讲就是负责保持集群中对象的实际状态与期望状态保持一致（用于调度程序以及节点状态检测）。
- **Etcd** ：负责存储集群中各种资源对象的信息。
- **Nodes**：构成了Kubernetes集群的集体计算能力，实际部署容器运行的地方。
- **Pods**：Kubernetes集群中资源的最小单位，可以包含一个或多个容器。
- **Kubelet** : 负责维护容器的生命周期，即通过控制docker，来创建、更新、销毁容器。
- **kube-proxy**：每个node节点都有一个组件kube-proxy，实际上是为service服务的，通过kube-proxy，实现流量从service到pod的转发，kube-proxy也可以实现简单的**负载均衡**功能。

Kubernetes 遵循非常传统的客户端/服务端的架构模式，客户端可以通过 RESTful 接口或者直接使用 kubectl 与 [Kubernetes](https://link.juejin.cn?target=http%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzI0MDQ4MTM5NQ%3D%3D%26mid%3D2247499493%26idx%3D3%26sn%3D4453fb452640045f0b7a2b020a4de61a%26chksm%3De9189bf9de6f12ef78a70fd575f1e76b377fd454539e94b9e869270a99fb08f48278154be2fa%26scene%3D21%23wechat_redirect) 集群进行通信，这两者在实际上并没有太多的区别，后者也只是对 Kubernetes 提供的 RESTful API 进行封装并提供出来。每一个 [Kubernetes](https://link.juejin.cn?target=http%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzI0MDQ4MTM5NQ%3D%3D%26mid%3D2247502359%26idx%3D1%26sn%3D8c16100c9731359b9864403183f44233%26chksm%3De918af0bde6f261da9f4960c0ed43e1552eeed9ff3ad0ba06bcfef75e3ba9e9d097e1c87a51c%26scene%3D21%23wechat_redirect) 集群都是由一组 Master 节点和一系列的 Worker 节点组成，其中 Master 节点主要负责存储集群的状态并为 Kubernetes 对象分配和调度资源。



下面，以部署一个nginx服务来说明kubernetes系统各个组件调用关系：

- 首先要明确，一旦kubernetes环境启动之后，master和node都会将自身的信息存储到etcd数据库中。

- 一个nginx服务的安装请求会首先被发送到master节点的apiServer组件。

- apiServer组件会调用scheduler组件来决定到底应该把这个服务安装到哪个node节点上。

- 在此时，它会从etcd中读取各个node节点的信息，然后按照一定的算法进行选择，并将结果告知apiServer。

- apiServer调用controller-manager去调度Node节点安装nginx服务。

- kubelet接收到指令后，会通知docker，然后由docker来启动一个nginx的pod，pod是kubernetes的最小操作单元，容器必须跑在pod中。

- 至此，一个nginx服务就运行了，如果需要访问nginx，就需要通过kube-proxy来对pod产生访问的代理。

这样，外界用户就可以访问集群中的nginx服务了。



## Service服务

在k8s集群中，service是一个抽象概念，它通过一个虚拟的IP映射指定的端口，将代理客户端发来的请求转到后端一组pod中的一个上。



### 背景

pod中的容器经常在不停地销毁和重建，因此pod的IP会不停的改变，如果客户端不知道pod的IP已经变化，继续访问之前的IP是无法访问容器。就算客户端知道了pod的IP变化了，每次重建后访问都需要更换IP，效率极为低下。



### service代理服务的作用

这时候就有了service代理服务，类似于nginx代理，作为客户端和pod的中间层，在通过service.yaml 创建service资源后，**随机虚拟创建一个固定的ip和port**，这时候再创建pod，比如deployment类型的nginx，service资源通过Label Selector（标签选择器）查找同名称空间下标签为刚才创建的deployment类型的名字为nginx资源进行绑定，就算pod不停地销毁和重建，并且不管有几个nginx的pod，都会跟这一个固定ip和port绑定，kubernetes内部会时刻更新这组关联关系，客户端访问固定ip和port，后端进行负载均衡机制分别访问不同的nginx的pod。pod的IP会不停的改变，但是service的ip和port不变。

![Pod、RC与Service的关系](https://raw.githubusercontent.com/OverCookkk/PicBed/master/blogImg/Pod%E3%80%81RC%E4%B8%8EService%E7%9A%84%E5%85%B3%E7%B3%BB.png)

（ReplicationController 能够动态地创建和销毁 Pod（例如，需要进行扩缩容，或者执行）,保证Service的服务能力和服务质量始终处于预期的标准）

>  Service的虚拟IP地址Cluster IP：外部网络无法ping通，只有kubernetes集群内部访问使用,但可以在各个node节点上直接通过ClusterIP:port访问。 





## deployment

Deployment是K8s用于管理Pod的资源对象，用来保证K8s中Pod的多实例、高可用与滚动更新、灰度部署等。可以说，Deployment是K8s中最常用最有用的一个对象，多用来发布无状态的应用。Deployment 其实并不是直接控制 Pod 的，而是借助 ReplicaSet 来控制 Pod 的。

### ReplicaSet

ReplicaSet 是 kubernetes 中的一种副本控制器资源，主要用于控制由其管理的Pod，使Pod副本的数量始终维持在预设的个数。

### Deployment的两大功能

- 水平扩展 / 收缩：Deployment Controller 只需要修改它所控制的 ReplicaSet 的 Pod 副本个数就可以了。比如，把这个值从 3 改成 4，那么 Deployment 所对应的 ReplicaSet，就会根据修改后的值自动创建一个新的 Pod。这就是“水平扩展”了；“水平收缩”则反之。

- 滚动更新：应用的发布和升级是运维最重要的核心职能之一，那么部署在K8S中的应用是怎么发布升级的呢？其实，你只要编辑修改下 Deployment 的 yaml 文件中镜像版本号，再 apply 一下，就更新成功了。



### Deployment常用字段解释

```yaml
apiVersion: apps/v1			# API群组和版本
kind: Deployment			# 资源类型
metadata:
  name <string>				# 资源名称，名称空间中要唯一
  namespace <string>		        # 名称空间
spec:
  minReadySeconds <integer>		# Pod就绪后多少秒后没有容器crash才视为“就绪”
  replicas <integer>			# Pod副本数，默认为1
  selector <object>			# 标签选择器，必须和template字段中Pod的标签一致
  template <object>			# 定义Pod模板

  revisionHistoryLimit <integer>	# 滚动更新历史记录数量，默认为10
  strategy <Object>			# 滚动更新策略
    type <string>			# 滚动更新类型，默认为RollingUpdate
    rollingUpdate <Object>		# 滚动更新参数，只能用于RollingUpdate类型
      maxSurge <string>			# 更新期间可以比replicas定义的数量多出的数量或比例
      maxUnavailable <string>		# 更新期间可以比replicas定义的数少的数量或比例 
  progressDeadlineSeconds <integer>	# 滚动更新故障超时时长，默认为600秒
  paused <boolean>			# 是否暂停部署
```



## ingress

要理解ingress，需要区分两个概念，ingress和ingress-controller：

**ingress对象：** 指的是k8s中的一个api对象，一般用yaml配置。作用是定义请求如何转发到service的规则，可以理解为配置模板。

**ingress-controller：** 具体实现反向代理及负载均衡的程序，对ingress定义的规则进行解析，根据配置的规则来实现请求转发。



### ingress工作原理

ingress controller通过和kubernetes api交互，动态的去感知集群中ingress规则变化，然后读取它，按照自定义的规则，规则就是写明了哪个域名对应哪个service，生成一段nginx配置，再写到nginx-ingress-controller的pod里，这个Ingress controller的pod里运行着一个Nginx服务，控制器会把生成的nginx配置写入/etc/nginx.conf文件中，然后reload一下使配置生效。以此达到域名分配置和动态更新的问题。



### ingress的实现方式

ingress-controller并不是k8s自带的组件，实际上ingress-controller只是一个统称，用户可以选择不同的ingress-controller实现，目前，由k8s维护的ingress-controller只有google云的GCE与ingress-nginx两个，其他还有很多第三方维护的ingress-controller。
