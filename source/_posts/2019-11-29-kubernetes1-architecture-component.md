---
title:      "【Kubernetes系列】第1篇 架构及组件介绍 "
date:       2019-11-29
banner_img: "https://image.my-blog.wang/header/header.jpg"
tags: [ Kubernetes ]
categories: [ Kubernetes ]

---

# 1、Kubernetes简介

Kubernetes是谷歌开源的容器集群管理系统，是Google多年大规模容器管理技术Borg的开源版本，主要功能包括:    

  * 基于容器的应用部署、维护和滚动升级

  * 负载均衡和服务发现

  * 跨机器和跨地区的集群调度

  * 自动伸缩

  * 无状态服务和有状态服务 

  * 广泛的Volume支持 

  * 插件机制保证扩展性

Kubernetes发展非常迅速，已经成为容器编排领域的领导者。

# 2、Kubernetes 架构及组件介绍

## 2.1 kubernetes 架构

Kubernetes架构如图所示：

![avatar](https://gitee.com/like-ycy/images/raw/master/blog/2019-11-29/2-1.jpg)

Kubernetes主要由以下几个核心组件构成：

  * etcd 保存整个集群的状态；

  * apiserver 提供了资源操作的唯一入口，并提供认证、授权、访问控制、API注册和发现等机制；

  * controller manager 负责维护集群的状态，比如故障检测、自动扩展、滚动更新等；

  * scheduler 负责资源的调度，按照预定的调度策略将实例（Pod）调度到相应的主机上；

  * kubelet 负责维护容器的生命周期，同时也负责存储卷和网络的管理；

  * container runtime 负责镜像管理以及容器的真正执行，在我们系统中指的是Docker

  * kube-proxy 负责为应用提供集群内部的服务发现和负载均衡

  * 推荐的插件

    * helm - kubernetes包管理工具

    * kube-dns/coreDNS 负责为整个集群提供DNS服务

    * Ingress Controller 为服务提供外网入口

    * Heapster 提供资源监控

    * Dashboard 提供GUI

    * Federation 提供跨可用区的集群

    * Fluentd-elasticsearch 提供集群日志采集、存储与查询

  

## 2.2 Kubernetes组件介绍

### 2.2.1 etcd

etcd是基于Raft一致性算法开发的分布式key-value存储，可用于服务发现、共享配置以及一致性保障（如数据库选主、分布式锁等）

   **etcd主要功能：**

  * 基本的key-value存储

  * 监听机制

  * key的过期及续约机制，用于监控和服务发现

  * 原子CAS和CAD，用于分布式锁和leader选举


 **Etcd基于RAFT的一致性**

 **leader节点选举方法**

  * 初始启动时，节点处于follower状态并被设定一个election timeout，如果在这一时间周期内没有收到来自leader的心跳检测，节点将发起选举，将自己切换为candidate（候选人）节点之后，向集群中的其他follow节点发送请求，询问其是否选举自己为leader

  * 当收到来自集群中过半数节点的接受投票后，节点即成为leader，开始接收保存client的数据并向其他的follower节点同步日志。如果没有达成一致，则candidate节点随机选择一个等待时间（150ms ～ 300ms）再次发起投票，得到集群中半数以上的follower接受的candidate将成为leader

  * leader节点依靠定时向follower节点发送心跳检测来保持其地位

  * 任何时候如果其他follower在election timeout期间没有收到来自leader的心跳检测，同样会将自己的状态切换为candidate并发起选举。每成功选举一次，新leader的步进数（Term）都会比之前leader的步进数加1 

**失效处理**

  * leader失效：其他没有收到心跳检测的节点将发起新的选举，当leader恢复后由于步进数小自动成为follower（日志会被新leader的日志覆盖）

  * follower节点不可用：follower节点不可用的情况相对比较容易解决。因为集群中的日志内容始终是从leader节点同步，只要这一节点再次加入集群时重新从leader节点处复制日志即可

  * 多个候选人（candidate）：冲突后candidate将随机选择一个等待时间（150ms ～ 300ms）再次发起投票，得到集群中半数以上的follower接受的candidate将成为leader  

讲到这里可能有同学发现Etcd和Zookeeper、Consul等一致性协议实现框架有些类似，的确这些中间件是比较类似的，关于其中的异同点，大家可以自行查阅资料。

  

### 2.2.2 kube-apiserver

kube-apiserver是Kubernetes最重要的核心组件之一，主要提供了如下功能：

  * 提供集群管理的REST API接口，包括认证授权、数据校验以及集群状态变更等

  * 提供同其他模块之间的数据交互(其他模块通过API Server查询或修改数据，只有API Server才直接操作etcd)

  

### 2.2.3 kube-scheduler

kube-scheduler负责分配调度Pod到集群内的节点上，它监听kube-
apiserver，查询还未分配Node的Pod，然后根据调度策略为这些Pod分配节点

通过以下三种方式可以指定Pod只运行在特定的Node节点上

  * nodeSelector:只调度到匹配指定label的Node上 

  * nodeAffinity:功能更丰富的Node选择器，比如支持集合操作 

  * podAffinity:调度到满足条件的Pod所在的Node上

  

### 2.2.4 kube-controller-manager

kube-controller-manager是Kubernetes的大脑，通过kube-
apiserver监控整个集群的状态，并确保集群处于预期的工作状态，它由一系列的控制器组成，这些控制器主要包括三组：

 **1\. 必须启动的控制器**  

  * eploymentController

  * DaemonSetController

  * NamesapceController

  * ReplicationController

  * RelicaSet

  * JobController

  * ...

 **2\. 默认启动的控制器**  

  * NodeController

  * ServiceController

  * PVBinderController

  * ...

 **3\. 默认禁止的可选控制器**

  * BootstrapSignerController

  * TokenCleanerController

  

### 2.2.5 Kubelet

每个Node节点上都运行一个kubelet守护进程，默认监听10250端口，接收并执行master发来的指令，管理Pod及Pod中的容器。每个kubelet进程会在API
Server上注册节点自身信息，定期向master节点汇报节点的资源使用情况

 **节点管理**

主要是节点自注册和节点状态更新:

  * Kubelet可以通过设置启动参数 --register-node 来确定是否向API Server注册自己; 

  * 如果Kubelet没有选择自注册模式，则需要用户自己配置Node资源信息，同时需要在Kubelet上配置集群中API Server的信息;

  * Kubelet在启动时通过API Server注册节点信息，并定时向API Server发送节点状态消息，API Server在接收到新消息后，将信息写入etcd  

 **容器健康检查**

Pod通过两类探针检查容器的健康状态

  * LivenessProbe 存活探针：通过该探针判断容器是否健康，告诉Kubelet一个容器什么时候处于不健康的状态。如果LivenessProbe探针探测到容器不健康，则kubelet将删除该容器，并根据容器的重启策略做相应的处理。如果一个容器不包含LivenessProbe探针，那么kubelet认为该容器的LivenessProbe探针返回的值永远是“Success”。

  * ReadinessProbe 就绪探针：用于判断容器是否启动完成且准备接收请求。如果 ReadinessProbe 探针探测到失败，则Pod的状态将被修改。Endpoint Controller将从Service的Endpoint中删除包含该容器所在Pod的IP地址的Endpoint条目。

以下是Pod的启动流程：

![avatar](https://gitee.com/like-ycy/images/raw/master/blog/2019-11-29/2-2-5.jpg)

  

### 2.2.6 kube-proxy

每台机器上都运行一个kube-proxy服务，它监听API
Server中service和Pod的变化情况，并通过userspace、iptables、ipvs等proxier来为服务配置负载均衡

代理模式（proxy-mode）提供如下三种类型：

  

 **1\. userspace**

![avatar](https://gitee.com/like-ycy/images/raw/master/blog/2019-11-29/2-2-6-1.png)

最早的负载均衡方案，它在用户空间监听一个端口，所有请求通过 iptables 转发到这个端口，然后在其内部负载均衡到实际的
Pod。service的请求会先从用户空间进入内核iptables，然后再回到用户空间（kube-proxy），由kube-
proxy完成后端Endpoints的选择和代理工作，这样流量从用户空间进出内核带来的性能损耗是不可接受的，所以产生了iptables的代理模式

  

 **2\. iptables:**

![avatar](https://gitee.com/like-ycy/images/raw/master/blog/2019-11-29/2-2-6-2.png)

iptables
mode完全使用iptables来完成请求过滤和转发。但是如果集群中存在大量的Service/Endpoint，那么Node上的iptables
rules将会非常庞大，添加或者删除iptables规则会引起较大的延迟。  

  

 **3\. ipvs:**

为了解决存在大量iptables规则时的网络延迟的问题，Kubernetes引入了ipvs的模式，（ipvs是LVS - Linux Virtual
Server 的重要组成部分，最早是由中国的章文嵩博士推出的一个开源项目，提供软件负载均衡的解决方案），下面是ipvs模式的原理图：

![avatar](https://gitee.com/like-ycy/images/raw/master/blog/2019-11-29/2-2-6-3.png)

