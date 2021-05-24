---
title:      " linux运维面试问题总结 "
date:       2020/06/29 9:21
# banner_img: "https://gitee.com/like-ycy/images/raw/master/blog/header.jpg"
tags: [ 面试 ]

---

### K8S集群组件及作用

- **API Server**：集群资源操作的唯一入口，结果存储在etcd中

- **etcd**：保存集群的配置信息和各种资源的状态信息
- **Controller Manager**：责管理集群各种资源，保证资源处于预期的状态，Controller Manager由多种controller组成，包括replication controller、endpoints controller、namespace controller、serviceaccounts controller等
- **Schedule**：资源调度，负责决定将Pod放到哪个Node上运行
- **Kubelet**：创建和运行schedule调度过来的Pod，并向master报告运行状态
- **Kube-proxy**：负责将访问的service的数据流转发到后端的容器
- 网络组件：flannel，calico

### K8S CNI组件及其原理

- **flannel**

Flannel 是一个overlay网络，工作在二层网络，flanneld进程会使用VXLAN协议把原始IP包加上目的MAC地址封装成二层数据帧，由linux内核把数据帧封装成UDP报文经过物理网络发送到另一台机器，然后另外一台机flanneld进行解包操作。

host-gw模式：直接把节点作为一个网关，比如节点node-02上的pod子网是10.64.1.0/24，那么集群中所有节点上都会增加一条路由指向node-02，要求宿主机在同一个二层网络

- **calico**

Calico是一个基于BGP的纯三层的网络方案，Calico在每个计算节点都利用Linux Kernel实现了一个高效的vRouter来负责数据转发。每个vRouter都通过BGP1协议把在本节点上运行的容器的路由信息向整个Calico网络广播，并自动设置到达其他节点的路由转发规则，没有额外的封包解包，能够节约CPU运算，提高网络效率。

### K8S pod之间无法通信，排查思路

跨主机pod之间无法通信：

- iptables问题，执行命令：

  ```
  iptables -P FORWARD ACCEPT  
  iptables -P INPUT ACCEPT
  ```

- docker配置问题，没连上flannel

### kube-proxy的三种代理模式及原理

- **userspace：**

  1、为每个service在node上打开一个随机端口（代理端口）

  2、建立iptables规则，将clusterip的请求重定向到代理端口

  3、到达代理端口（用户空间）的请求再由kubeproxy转发到后端pod。

  这里为什么需要建iptables规则，因为kube-proxy 监听的端口在用户空间，所以需要一层 iptables 把访问服务的连接重定向给 kube-proxy 服务，这里就存在内核态到用户态的切换，代价很大，因此就有了iptables。

- **iptables：**

  iptables由kube-proxy动态的管理，kube-proxy不再负责转发，数据包的走向完全由iptables规则决定，这样的过程不存在内核态到用户态的切换，效率明显会高很多。但是随着service的增加，iptables规则会不断增加，导致内核十分繁忙

- **ipvs：**

  用ipset存储iptables规则，这样规则的数量就能够得到有效控制，而在查找时就类似hash表的查找

### kube-proxy挂掉的影响

service的访问请求将不会转发到Pod上，用户无法访问

### k8s service涉及到的组件

kube-proxy、pod、Replication Controller

### ingress和service的区别

Service可以看作是一组提供相同服务的Pod对外的访问接口。借助Service，应用可以方便地实现服务发现和负载均衡。Ingress 是全局的，为了代理不同后端 Service 而设置的负载均衡服务。

Service 有三种对外暴露的方法，但是由于每个 Service 都要有一个负载均衡的服务，所以采用 Service 的话，会造成既浪费成本又高的现象。而ingress是一个全局的负载均衡器,然后我只需要通过访问 URL 就可以把请求转发给不同的后端 Service ，从而可以访问到界面，而不是每个 Service 都需要负载均衡。

### k8s设置node节点污点

kubectl taint node [node] key=value[effect]   
     其中[effect] 可取值: [ NoSchedule | PreferNoSchedule | NoExecute ]
      NoSchedule: 一定不能被调度
      PreferNoSchedule: 尽量不要调度
      NoExecute: 不仅不会调度, 还会驱逐Node上已有的Pod

### k8s的pause容器有什么用

pod内的其他容器会共用pause容器的网络栈和存储卷，保证pod内的其他容器的端口不能冲突，彼此都是通过localhost就可以访问，扮演PID1的角色,并在子进程称为"孤儿进程"的时候,通过调用wait()收割这个子进程,这样就不用担心我们的Pod的PID namespace里会堆满僵尸进程了。

### kubernetes中的pause容器主要为每个业务容器提供以下功能：

PID命名空间：Pod中的不同应用程序可以看到其他应用程序的进程ID。

网络命名空间：Pod中的多个容器能够访问同一个IP和端口范围。

IPC命名空间：Pod中的多个容器能够使用SystemV IPC或POSIX消息队列进行通信。

UTS命名空间：Pod中的多个容器共享一个主机名；Volumes（共享存储卷）：

Pod中的各个容器可以访问在Pod级别定义的Volumes。

### K8S PLEG错误处理

**问题**：k8s node节点显示 no ready

**过程1**：pleg是pod生命周期事件生成器，pleg在每次迭代检查中会 调用docker ps来检测容器状态的变化，并调用docker Inspect来获取这些容器的详细信息。在完成每次迭代之后，它更新一个时间戳。如果时间戳有一段时间没有更新(即3分钟)，则运行状况检查失败。

**过程2**：执行docker ps果然很慢

**解决**：因为是测试环境，跑了太多的pod，机器资源已被占用满，重启了机器，执行systemctl daemon-reexec，

临时解决了，systemctl的一个小bug。

### Mysql 主从不同步检查思路

1、master，slave节点个执行show master status查看，如果有NO，重新做主从

2、也可能是在slave上进行了写操作，或者slave机器重启，或者事务回滚

3、解决方法：停掉主从同步，忽略一次错误，再开启同步：

### Mysql备份方案

每三天做全量备份，中间每天做增量，增量根据position值或者时间点进行备份

### nginx状态码

301 永久重定向

302 临时重定向

403 服务器拒绝请求

404 访问的资源不存咋

500 服务器内部错误

502 错误网关

503 服务不可达

### nginx获取用户真实ip

```
proxy_set_header   Host             $host;
proxy_set_header   X-Real-IP        $remote_addr;
proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
set_real_ip_from 0.0.0.0/0;         # 额外增加的配置            
real_ip_header  X-Forwarded-For;    # 额外增加的配置            
real_ip_recursive   on;             # 额外增加的配置
```

### 端口time_wait解决

```
net.ipv4.tcp_syncookies = 1 表示开启SYN Cookies。当出现SYN等待队列溢出时，启用cookies来处理，可防范少量SYN攻击，默认为0，表示关闭；
net.ipv4.tcp_tw_reuse = 1 表示开启重用。允许将TIME-WAIT sockets重新用于新的TCP连接，默认为0，表示关闭；
net.ipv4.tcp_tw_recycle = 1 表示开启TCP连接中TIME-WAIT sockets的快速回收，默认为0，表示关闭。
net.ipv4.tcp_fin_timeout 修改系默认的 TIMEOUT 时间
```

### linux系统执行命令很卡

- top命令查看系统资源使用情况，cpu、内存、负载
- iostat命令查看磁盘IO使用情况

### shell脚本的变量

$0  文件名及路径

$1,$2  参数1，参数2

$#  传递给脚本或函数的参数个数

$$  当前Shell进程ID

$?  判断上个命令的执行成功与否，0为成功。

$@  传递脚本或函数的所有参数

$*  传递脚本或函数的所有参数

### linux cpu 负载概念

这些数据来自于文件/proc/loadavg，内核会负责统计出这些数据。
top和uptime命令显示的内容就来自于这个文件，根据proc的帮助文件可知，这里的值就是单位时间内处于运行状态以及等待磁盘 I/O状态的平均job数量。这里的运行状态和job都是内核的概念，这里进行说明：
1、 对于内核而言，进程和线程都是job
2、 job处于运行状态指job处于内核的运行队列中，正在或等待被CPU调度（用户空间的进程正在运行不代表需要被CPU调度，有可能在等待I/O，也有可能在sleep等等）

### linux硬链接和软链接原理

- 硬链接：在Linux系统中，多个文件名指向同一索引节点(Inode)是正常且允许的。一般这种链接就称为硬链接。硬链接的作用之一是允许一个文件拥有多个有效路径名，这样用户就可以建立硬链接到重要的文件，以防止“误删”源数据。
- 软链接： 软链接就是一个普通文件，只是数据块内容有点特殊，文件用户数据块中存放的内容是另一文件的路径名的指向，通过这个方式可以快速定位到软连接所指向的源文件实体。软链接可对文件或目录创建。

软连接和硬链接的特点：

软链接：

- 1.软链接是存放另一个文件的路径的形式存在。
- 2.软链接可以 跨文件系统 ，硬链接不可以。
- 3.软链接可以对一个不存在的文件名进行链接，硬链接必须要有源文件。
- 4.软链接可以对目录进行链接。

硬链接：

- 1. 硬链接，以文件副本的形式存在。但不占用实际空间。
- 2. 不允许给目录创建硬链接。
- 3. 硬链接只有在同一个文件系统中才能创建。
- 4. 删除其中一个硬链接文件并不影响其他有相同 inode 号的文件。

### linux进程退出指令

 INT（快速关闭）----是当用户键入<Control-C>时由终端驱动程序发送的信号。这是一个终止当前操作的请求，如果捕获了这个信号，一些简单的程序应该退出，或者允许自给被终止，这也是程序没有捕获到这个信号时的默认处理方法。拥有命令行或者输入模式的那些程序应该停止它们在做的事情，清除状态，并等待用户的再次输入。

  TERM（快速关闭）----是请求彻底终止某项执行操作，它期望接收进程清除自给的状态并退出。

  HUP---- 平滑启动。如果想要更改配置而不需停止并重新启动服务，请使用该命令。在对配置文件作必要的更改后，发出该命令以动态更新服务配置。

  QUIT：从容关闭。

