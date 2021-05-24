---
title:      " 运维面试题系列--Kubernetes "
date:       2021-03-03
banner_img: "https://gitee.com/like-ycy/images/raw/master/blog/header.jpg"
tags: [ 面试 ]


---

# Kubernetes面试问题



## 理论部分

### 1、**kubernetes是什么？**

Kubernetes 是一个跨主机集群的开源的容器调度平台，它可以自动化应用容器的部署、扩展和操作, 提供以容器为中心的基础架构。

### 2、k8s前身是谷歌的borg系统

### 3、**k8s可以干什么？**

- 快速部署应用
- 快速扩展应用
- 无缝对接新的应用功能
- 节省资源，优化硬件资源的使用

###   4、**Kubernetes 具有如下特点:**

- 便携性: 无论公有云、私有云、混合云还是多云架构都全面支持
- 可扩展: 它是模块化、可插拔、可挂载、可组合的，支持各种形式的扩展
- 自修复: 它可以自保持应用状态、可自重启、自复制、自缩放的，通过声明式语法提供了强大的自修复能力

### 5、k8s抽象对象

pod、service、volum、job、replica set、deployment、secret、namespace ...

### 6、K8s集群初始有两个名称空间

kube-system、default

### 7、k8s滚动升级

滚动升级一个服务，实际是创建一个新的RS，然后逐渐将新RS中副本数增加到理想状态，将旧RS中的副本数减小到0的复合操作 
可以使用kubectl  scale命令

### 8、Kubernetes主要由以下几个核心组件组成：

- master节点
  - etcd保存了整个集群的状态;
  - apiserver提供了资源操作的唯一入口，并提供认证、授权、访问控制、API注册和发现等机制；
  - controller manager负责维护集群的状态，比如故障检测、自动扩展、滚动更新等；
  - scheduler负责资源的调度，按照预定的调度策略将Pod调度到相应的机器上；
- node节点：
	- kubelet负责维护容器的生命周期，同时也负责Volume（CVI）和网络（CNI）的管理；
	- Container runtime负责镜像管理以及Pod和容器的真正运行（CRI）；
	- kube-proxy负责为Service提供cluster内部的服务发现和负载均衡；

除了核心组件，还有一些推荐的Add-ons：

- coredns负责为整个集群提供DNS服务
-  Ingress Controller为服务提供外网入口
- Dashboard提供GUI

### 9、service四种type

- ClusterIP（默认） - 在集群中内部IP上暴露服务。此类型使Service只能从群集中访问。
- NodePort - 通过每个 Node 上的 IP 和静态端口（NodePort）暴露服务。NodePort 服务会路由到 ClusterIP 服务，这个ClusterIP 服务会自动创建。通过请求 <NodeIP>:<NodePort>，可以从集群的外部访问一个 NodePort 服务。
- LoadBalancer - 使用云提供商的负载均衡器（如果支持），可以向外部暴露服务。外部的负载均衡器可以路由到 NodePort 服务和 ClusterIP 服务。 
- ExternalName - 通过返回 CNAME 和它的值，可以将服务映射到 externalName 字段的内容，没有任何类型代理被创建。

### 10、volum支持类型

hostPath、nfs、glusterfs、rbd、cephfs、local

### 11、k8s Controller Manager

- Replication Controler      			   保证pod副本数
- Node controller		       		        通过API server定时获取Node的相关信息，实现管理和监控Node
- ResourceQuota Controller        	资源配额管理确保指定的资源对象在任何时候都不会超量占用系统物理资源
- Namespace Controller		 		  Namespace Controller定时通过API Server读取用户创建的保存在etcd的namespace
- endpoint controller      		 	     负责生成和维护所有Endpoints对象的控制器
- service controller                            监听Service变化 
- serviceAccount controller   
- Token controller 

### 12、k8s pod之间的通信

- 同一个Pod中容器之间的通信
这种场景对于Kubernetes来说没有任何问题，根据Kubernetes的架构设计。Kubernetes创建Pod时，首先会创建一个pause容器，为Pod指派一个唯一的IP地址。然后，以pause的网络命名空间为基础，创建同一个Pod内的其它容器（–net=container:xxx）。因此，同一个Pod内的所有容器就会共享同一个网络命名空间，在同一个Pod之间的容器可以直接使用localhost进行通信。

- 不同Pod中容器之间的通信
对于此场景，情况现对比较复杂一些，这就需要解决Pod间的通信问题。在Kubernetes通过flannel、calic等网络插件解决Pod间的通信问题。本文以flannel为例说明在Kubernetes中网络模型，flannel是kubernetes默认提供网络插件。Flannel是由CoreOs团队开发社交的网络工具，CoreOS团队采用L3 Overlay模式设计flannel， 规定宿主机下各个Pod属于同一个子网，不同宿主机下的Pod属于不同的子网。
flannel会在每一个宿主机上运行名为flanneld代理，其负责为宿主机预先分配一个子网，并为Pod分配IP地址。Flannel使用Kubernetes或etcd来存储网络配置、分配的子网和主机公共IP等信息。数据包则通过VXLAN、UDP或host-gw这些类型的后端机制进行转发。

### 13、k8s handless service

Headless 服务即不需要 Cluster IP 的服务，即在创建服务的时候指定 `spec.clusterIP=None`。包括两种类型

- 不指定 Selectors，但设置 externalName，即上面的（2），通过 CNAME 记录处理
- 指定 Selectors，通过 DNS A 记录设置后端 endpoint 列表

### 14、kube-proxy的三种代理模式

- userspace：最早的负载均衡方案，它在用户空间监听一个端口，所有服务通过 iptables 转发到这个端口，然后在其内部负载均衡到实际的 Pod。该方式最主要的问题是效率低，有明显的性能瓶颈。
- iptables：目前推荐的方案，完全以 iptables 规则的方式来实现 service 负载均衡。该方式最主要的问题是在服务多的时候产生太多的 iptables 规则，非增量式更新会引入一定的时延，大规模情况下有明显的性能问题
- ipvs：为解决 iptables 模式的性能问题，v1.11 新增了 ipvs 模式（v1.8 开始支持测试版，并在 v1.11 GA），采用增量式更新，并可以保证 service 更新期间连接保持不断开

注意：使用 ipvs 模式时，需要预先在每台 Node 上加载内核模块 `nf_conntrack_ipv4`, `ip_vs`, `ip_vs_rr`, `ip_vs_wrr`, `ip_vs_sh` 等。IPVS 模式也会使用 iptables 来执行 SNAT 和 IP 伪装（MASQUERADE），并使用 ipset 来简化 iptables 规则的管理。

### 15、kube-proxy的不足

kube-proxy 目前仅支持 TCP 和 UDP，不支持 HTTP 路由，并且也没有健康检查机制。这些可以通过自定义 [Ingress Controller](https://feisky.gitbooks.io/kubernetes/content/plugins/ingress.html) 的方法来解决。

### 16、k8s pod内的pause容器的作用

pod内的其他容器会共用pause容器的网络栈和存储卷，保证pod内的其他容器的端口不能冲突，彼此都是通过localhost就可以访问，扮演PID1的角色,并在子进程称为”孤儿进程”的时候,通过调用wait()收割这个子进程,这样就不用担心我们的Pod的PID namespace里会堆满僵尸进程了。

**pause容器主要为每个业务容器提供以下功能：**

PID命名空间：Pod中的不同应用程序可以看到其他应用程序的进程ID。

网络命名空间：Pod中的多个容器能够访问同一个IP和端口范围。

IPC命名空间：Pod中的多个容器能够使用SystemV IPC或POSIX消息队列进行通信。

UTS命名空间：Pod中的多个容器共享一个主机名；Volumes（共享存储卷）

### 17、k8s pod init容器

- Init 容器在所有容器运行之前执行（run-to-completion），常用来初始化配置。

- 如果为一个 Pod 指定了多个 Init 容器，那这些init容器会按顺序一次运行一个。 每个 Init 容器必须运行成功，下一个才能够运行。 当所有的 Init 容器运行完成时，Kubernetes 初始化 Pod 并像平常一样运行应用容器。

### 18、k8s service和ingress的区别

Service 虽然解决了服务发现和负载均衡的问题，但它在使用上还是有一些限制，比如

－ 只支持 4 层负载均衡，没有 7 层功能 － 对外访问的时候，NodePort 类型需要在外部搭建额外的负载均衡，而 LoadBalancer 要求 kubernetes 必须跑在支持的 cloud provider 上面

Ingress 就是为了解决这些限制而引入的新资源，主要用来将服务暴露到 cluster 外面，并且可以自定义服务的访问策略