---
title:      "【Kubernetes系列】第9篇 网络原理解析"
date:       2019-12-11
banner_img: "https://image.my-blog.wang/header/header.jpg"
tags: [ Kubernetes ]

---

# 1、Linux网络基础

**Network Namespace(网络命名空间)：**

Linux在网络栈中引入网络命名空间，将独立的网络协议栈隔离到不同的命令空间中，彼此间无法通信；docker利用这一特性，实现不容器间的网络隔离。

**Iptables/Netfilter：**

Netfilter负责在内核中执行各种挂接的规则(过滤、修改、丢弃等)，运行在内核模式中；Iptables模式是在用户模式下运行的进程，负责协助维护内核中Netfilter的各种规则表；通过分析进入主机的网络封包，将封包的表头数据提取出来进行分析，以决定该联机为放行或抵挡的机制。由于这种方式可以直接分析封包表头数据，所以包括硬件地址(MAC), 软件地址 (IP), TCP, UDP, ICMP 等封包的信息都可以进行过滤分析。

**Veth设备对：**

Veth设备对的引入是为了实现在不同网络命名空间的通信。

**Bridge(网桥)：**

网桥是一个二层网络设备，是最简单的CNI网络插件，它首先在Host创建一个网桥，然后再通过veth pair连接该网桥到container netns。另外，Bridge模式下，多主机网络通信需要额外配置主机路由。

**Route(路由)：**

Linux系统包含一个完整的路由功能，当IP层在处理数据发送或转发的时候会使用路由表来决定发往哪里。

**Container Network Interface (CNI) ：**

最早是由CoreOS发起的容器网络规范，是 Kubernetes网络插件的基础。其基本思想为:Container Runtime在创建容器时，先创建好network namespace，然后调用CNI插件为这个netns配置网络，其后再启动容器内的进程。

# 2、K8S网络模型

Kubernetes网络有一个重要的基本设计原则：**每个Pod拥有唯一的IP**。

这个Pod IP被该Pod内的所有容器共享，并且其它所有Pod都可以路由到该Pod。你可曾注意到，你的Kubernetes节点上运行着一些"pause"容器？它们被称作“沙盒容器（sandbox containers）"，其唯一任务是保留并持有一个网络命名空间（netns），该命名空间被Pod内所有容器共享。通过这种方式，即使一个容器死掉，新的容器创建出来代替这个容器，Pod IP也不会改变。这种IP-per-pod模型的巨大优势是，Pod和底层主机不会有IP或者端口冲突。我们不用担心应用使用了什么端口。

这点满足后，Kubernetes唯一的要求是，这些Pod IP可被其它所有Pod访问，不管那些Pod在哪个节点。

##  2.1 节点内通信

第一步是确保同一节点上的Pod可以相互通信，然后可以扩展到跨节点通信、internet上的通信，等等。

![avatar](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-11/2-2-1-a.png)

在每个Kubernetes节点（本场景指的是Linux机器）上，都有一个根（root）命名空间（root是作为基准，而不是超级用户）-- root netns(root network namespace)。

主要的网络接口 eth0 就是在这个root netns下。

![avatar](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-11/2-2-1-b.png)

类似的，每个Pod都有其自身的netns(network namespace)，通过一个虚拟的以太网对连接到root netns。这基本上就是一个管道对，一端在root netns内，另一端在Pod的netns内。

我们把Pod端的网络接口叫 eth0，这样Pod就不需要知道底层主机，它认为它拥有自己的根网络设备。另一端命名成比如 vethxxx。你可以用 ifconfig 或者 ip a 命令列出你的节点上的所有这些接口。

![avatar](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-11/2-2-1-c.png)

节点上的所有Pod都会完成这个过程。这些Pod要相互通信，就要用到linux的以太网桥 cbr0 了。Docker使用了类似的网桥，称为docker0。你可以用 brctl show 命令列出所有网桥。

![avatar](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-11/2-2-1-d.gif)

假设一个网络数据包要由pod1到pod2

1) 它由pod1中netns的eth0网口离开，通过vethxxx进入root netns。

2) 然后被传到cbr0，cbr0使用ARP请求，说“谁拥有这个IP”，从而发现目标地址。

3）vethyyy说它有这个IP，因此网桥就知道了往哪里转发这个包。

4）数据包到达vethyyy，跨过管道对，到达pod2的netns。

这就是同一节点内容器间通信的流程。当然也可以用其它方式，但是无疑这是最简单的方式，同时也是Docker采用的方式。

## 2.2 不同节点间通信

正如我前面提到，Pod也需要跨节点可达。Kubernetes不关心如何实现。我们可以使用L2（ARP跨节点），L3（IP路由跨节点，就像云提供商的路由表），Overlay网络，或者甚至信鸽。无所谓，只要流量能到达另一个节点的期望Pod就好。每个节点都为Pod IPs分配了唯一的CIDR块（一段IP地址范围），因此每个Pod都拥有唯一的IP，不会和其它节点上的Pod冲突。

大多数情况下，特别是在云环境上，云提供商的路由表就能确保数据包到达正确的目的地。我们在每个节点上建立正确的路由也能达到同样的目的。许多其它的网络插件通过自己的方式达到这个目的。

这里我们有两个节点，与之前看到的类似。每个节点有不同的网络命名空间、网络接口以及网桥。

![avatar](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-11/2-2-2-a.gif)

假设一个数据包要从pod1到达pod4（在不同的节点上）

1）它由pod1中netns的eth0网口离开，通过vethxxx进入root netns。

2）然后被传到cbr0，cbr0通过发送ARP请求来找到目标地址。

3）本节点上没有Pod拥有pod4的IP地址，根据路由判断数据包由cbr0 传到主网络接口 eth0。

4）数据包的源地址为pod1，目标地址为pod4，它以这种方式离开node1进入电缆。

5）路由表有每个节点的CIDR块的路由设定，它把数据包路由到CIDR块包含pod4的IP的节点。

6）因此数据包到达了node2的主网络接口eth0。现在即使pod4不是eth0的IP，数据包也仍然能转发到cbr0，因为节点配置了IP forwarding enabled。节点的路由表寻找任意能匹配pod4 IP的路由。它发现了 cbr0 是这个节点的CIDR块的目标地址。你可以用route -n命令列出该节点的路由表，它会显示cbr0的路由，类型如下：

![avatar](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-11/2-2-2-b.png)

7）网桥接收了数据包，发送ARP请求，发现目标IP属于vethyyy。

8）数据包跨过管道对到达pod4。

# 3、覆盖网络

覆盖⽹络(overlay network)是将TCP数据包装在另⼀种⽹络包⾥⾯进⾏路由转发和通信的技术。Overlay⽹络不是默认必须的，但是它们在特定场景下⾮常有⽤。⽐如当我们没有⾜够的IP空间，或者⽹络⽆法处理额外路由，抑或当我们需要Overlay提供的某些额外管理特性。⼀个常⻅的场景是当云提供商的路由表能处理的路由数是有限制时，例如AWS路由表最多⽀持50条路由才不⾄于影响⽹络性能。因此如果我们有超过50个Kubernetes节点， AWS路由表将不够。这种情况下，使⽤Overlay⽹络将帮到我们。

本质上来说， Overlay就是在跨节点的本地⽹络上的包中再封装⼀层包。你可能不想使⽤Overlay⽹络，因为它会带来由封装和解封所有报⽂引起的时延和复杂度开销。通常这是不必要的，因此我们应当在知道为什么我们需要它时才使⽤它。

为了理解Overlay⽹络中流量的流向，我们拿Flannel做例⼦，它是CoreOS 的⼀个开源项⽬。Flannel通过给每台宿主机分配⼀个⼦⽹的⽅式为容器提供虚拟⽹络，它基于Linux TUN/TAP，使⽤UDP封装IP包来创建overlay⽹络，并借助etcd维护⽹络的分配情况。

![avatar](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-11/3-1.gif)

这⾥我们注意到它和之前我们看到的设施是⼀样的，只是在root netns中新增了⼀个虚拟的以太⽹设备，称为flannel0。它是虚拟扩展⽹络Virtual Extensible LAN（VXLAN）的⼀种实现，但是在Linux上，它只是另⼀个⽹络接⼝。

从pod1到pod4（在不同节点）的数据包的流向类似如下：

1）它由pod1中netns的eth0⽹⼝离开，通过vethxxx进⼊root netns。
2）然后被传到cbr0， cbr0通过发送ARP请求来找到⽬标地址。
3）数据封装

  3a. 由于本节点上没有Pod拥有pod4的IP地址，因此⽹桥把数据包发送给了flannel0，因为节点的路由表上flannel0被配成了Pod⽹段的⽬标地址。

  3b. flanneld daemon和Kubernetes apiserver或者底层的etcd通信，它知道所有的Pod IP，并且知道它们在哪个节点上。因此Flannel创建了Pod IP和Node IP之间的映射（在⽤户空间）。flannel0取到这个包，并在其上再⽤⼀个UDP包封装起来，该UDP包头部的源和⽬的IP分别被改成了对应节点的IP，然后发送这个新包到特定的VXLAN端⼝（通常是8472）。

![avatar](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-11/3-2.png)

尽管这个映射发⽣在⽤户空间，真实的封装以及数据的流动发⽣在内核空间，因此仍然是很快的。  

  3c. 封装后的包通过eth0发送出去，因为它涉及了节点间的路由流量。

\4. 包带着节点IP信息作为源和⽬的地址离开本节点。
\5. 云提供商的路由表已经知道了如何在节点间发送报⽂，因此该报⽂被发送到⽬标地址node2。
\6. 数据解包

  6a. 包到达node2的eth0⽹卡，由于⽬标端⼝是特定的VXLAN端⼝，内核将报⽂发送给了
flannel0。
  6b. flannel0解封报⽂，并将其发送到 root 命名空间下。从这⾥开始，报⽂的路径和我们之前在Part1 中看到的⾮Overlay⽹络就是⼀致的了。
  6c. 由于IP forwarding开启着，内核按照路由表将报⽂转发给了cbr0。

\7. ⽹桥获取到了包，发送ARP请求，发现⽬标IP属于vethyyy。
\8. 包跨过管道对到达pod4

这就是Kubernetes中Overlay⽹络的⼯作⽅式，虽然不同的实现还是会有细微的差别。**有个常⻅的误区是，当我们使⽤Kubernetes，我们就不得不使⽤Overlay⽹络。事实是，这完全依赖于特定场景。因此请确保在确实需要的场景下才使⽤**。

# 4、动态集群

由于Kubernetes（更通⽤的说法是分布式系统）天⽣具有不断变化的特性，因此它的Pod（以及Pod的IP）总是在改变。变化的原因可以是针对不可预测的Pod或节点崩溃⽽进⾏的滚动升级和扩展。这使得Pod IP不能直接⽤于通信。

我们看⼀下Kubernetes Service，它是⼀个虚拟IP，并伴随着⼀组Pod IP作为Endpoint（通过标签选择器识别）。它们充当虚拟负载均衡器，其IP保持不变，⽽后端Pod IP可能会不断变化。

```yaml
kind: Service
apiVersion: V1
metadata:
  name: svc2
spec:
  type: ClusterIP
  selector:
    app: myapp
  ClusterIP: 10.200.100.214
  ports:
  - name: http
    port: 80
```

整个虚拟IP的实现实际上是⼀组iptables（最新版本可以选择使⽤IPVS，但这是另⼀个讨论）规则，由Kubernetes组件kube-proxy管理。这个名字现在实际上是误导性的。它在v 1.0之前确实是⼀个代理，并且由于其实现是内核空间和⽤户空间之间的不断复制，它变得⾮常耗费资源并且速度较慢。现在，它只是⼀个控制器，就像Kubernetes中的许多其它控制器⼀样，它watch api serverendpoint的更改并相应地更新iptables规则。

![avatar](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-11/4-0-a.png)

有了这些iptables规则，每当数据包发往Service IP时，它就进⾏DNAT（DNAT=⽬标⽹络地址转换）操作，这意味着⽬标IP从Service IP更改为其中⼀个Endpoint - Pod IP - 由iptables随机选择。这可确保负载均匀分布在后端Pod中。

![avatar](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-11/4-0-b.png)

当这个DNAT发⽣时，这个信息存储在conntrack中——Linux连接跟踪表（iptables规则5元组转译并完成存储：protocol， srcIP， srcPort， dstIP， dstPort）。这样当请求回来时，它可以un-DNAT，这意味着将源IP从Pod IP更改为Service IP。这样，客户端就不⽤关⼼后台如何处理数据包流。

因此通过使⽤Kubernetes Service，我们可以使⽤相同的端⼝⽽不会发⽣任何冲突（因为我们可以将端⼝重新映射到Endpoint）。这使服务发现变得⾮常容易。我们可以使⽤内部DNS并对服务主机名进⾏硬编码。

我们甚⾄可以使⽤Kubernetes提供的service主机和端⼝的环境变量来完成服务发现。

**专家建议：**采取第⼆种⽅法，你可节省不必要的DNS调⽤，但是由于环境变量存在创建顺序的局限性（环境变量中不包含后来创建的服务），推荐使⽤DNS来进⾏服务名解析。

## 4.1 出站流量

到⽬前为⽌我们讨论的Kubernetes Service是在⼀个集群内⼯作。但是，在⼤多数实际情况中，应⽤程序需要访问⼀些外部api/website。

通常，节点可以同时具有私有IP和公共IP。对于互联⽹访问，这些公共和私有IP存在某种1：1的NAT，特别是在云环境中。

对于从节点到某些外部IP的普通通信，源IP从节点的专⽤IP更改为其出站数据包的公共IP，⼊站的响应数据包则刚好相反。但是，当Pod发出与外部IP的连接时，源IP是Pod IP，云提供商的NAT机制不知道该IP。因此它将丢弃具有除节点IP之外的源IP的数据包。

因此你可能也猜对了，我们将使⽤更多的iptables！这些规则也由kube-proxy添加，执⾏SNAT（源⽹络地址转换），即IP MASQUERADE（IP伪装）。它告诉内核使⽤此数据包发出的⽹络接⼝的IP，代替源Pod IP同时保留conntrack条⽬以进⾏反SNAT操作。

## 4.2 入站流量

到⽬前为⽌⼀切都很好。Pod可以互相交谈，也可以访问互联⽹。但我们仍然缺少关键部分 - 为⽤户请求流量提供服务。截⾄⽬前，有两种主要⽅法可以做到这⼀点：

- **NodePort /云负载均衡器（L4 - IP和端⼝）**

将服务类型设置为NodePort默认会为服务分配范围为30000-33000d的nodePort。即使在特定节点上没有运⾏Pod，此nodePort也会在每个节点上打开。此NodePort上的⼊站流量将再次使⽤iptables发送到其中⼀个Pod（该Pod甚⾄可能在其它节点上！）。

云环境中的LoadBalancer服务类型将在所有节点之前创建云负载均衡器（例如ELB），命中相同的nodePort。

- **Ingress（L7 - HTTP / TCP）**

许多不同的⼯具，如Nginx， Traefik， HAProxy等，保留了http主机名/路径和各⾃后端的映射。通常这是基于负载均衡器和nodePort的流量⼊⼝点，其优点是我们可以有⼀个⼊⼝处理所有服务的⼊站流量，⽽不需要多个nodePort和负载均衡器。

## 4.3 网站策略

可以把它想象为Pod的安全组/ ACL。NetworkPolicy规则允许/拒绝跨Pod的流量。确切的实现取决于⽹络层/CNI，但⼤多数只使⽤iptables。

