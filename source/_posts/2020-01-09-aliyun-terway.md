---
title:      " 阿里云terway源码分析 "
date:       2020-01-09
banner_img: "https://image.my-blog.wang/header/header.jpg"
tags: [ 阿里云 ]

---

# 背景

随着公司业务的发展，底层容器环境也需要在各个区域部署，实现多云架构， 使用各个云厂商提供的CNI插件是k8s多云环境下网络架构的一种高效的解法。我们在阿里云的方案中，便用到了阿里云提供的CNI插件terway。terway所提供的VPC互通的网络方案，方便对接已有的基础设施，同时没有overlay网络封包解包的性能损耗，简单易用，出现网络问题方便诊断。本文对该插件做简单的代码分析，理解其原理，以便后期诊断问题和维护。

# 功能划分

阿里云开源的terway代码有三部分组成:

- CNI plugin： 即CNI插件，实现`ADD、DEL、VERSION`三个接口来供kubelet调用， 该插件将kubelet传递的参数进行简单处理后，会通过gRPC调用terwayBackendServer来实现具体的逻辑，例如申请网络设备等。同步调用terwayBackendServer将网络设备分配完毕之后，会通过`ipvlanDriver.Driver`进行pod sandbox network namespace的`Setup`操作，同时还会通过TC进行流控。该插件会通过daemonSet中initContainer安装到所有node上。
- backend server： terway中主要的执行逻辑， 会进行IPAM管理，并申请对应的网络设备， 这部分是本次着重分析的对象。该程序以daemonSet的方式运行在每个节点上。
- networkPolicy： 该部分是借助calico felix实现， 完全与上面两部分解耦。我们看到terway创建的网络设备是以cali为前缀的， 其实就是为了兼容calico的schema。

# TerwayBackendServer

在terway的main函数中会启动gRPC server监听请求，同时会创建一个`TerwayBackendServer`， TerwayBackendServer封装全部操作逻辑，在`newNetworkService`函数中会依次初始化各个子模块实例，具体包括：

- ECS client　用来操作ECS client, 所有创建删除更新操作最后都会通过该client进行处理，简单封装了一层alicloud的SKD
- kubernetes pod 管理模块，用来同步kubernetes pod信息
- resouceDB 用来存储状态信息，便于重启等操作后恢复状态
- resourceManager 管理资源分配的实例，terway会根据不同的配置生成不同的resourceManager，此处我们使用的是`ENIMultiIP`这种模式，对应的就是`newENIIPResourceManager`

ENIMultiIP模式会申请阿里云弹性网卡并配置多个辅助VPC的IP地址，将这些辅助IP地址映射和分配到Pod中，这些Pod的网段和宿主机网段是一致的，能够实现VPC网络互通。

整个架构如下图所示：

![twrway.png](https://gitee.com/like-ycy/images/raw/master/blog/2020-01-09/terway.png)

首先我们理解一下kubernetes pod管理模块，该模块用于获取kubernetes pod状态。terway为了支持一些高级的特性，例如流控等，有一些信息无法通过CNI调用传递过来，　还是得去kubernetes中去查询这些信息。此外CNI调用在一些异常情况下可能无法准确回调CNI插件， 例如用户直接`kubectl delete pod --force --graceperiod=0`，此时就需要kubernetes作为唯一的`single source of truth`， 保证最后网络设备在pod删除时肯定能够被释放掉。 它内部主要的方法就是`GetPod`与`GetLocalPod`。`GetPod`方法会请求apiserver返回pod信息，如果该pod已经在apiserver中删除，就会从本地的storage中获取。该storage是用boltDB做为底层存储的一个本地文件，每个被处理过的pod都会在该storage中保存一份信息，且该pod副本并不会随着apiserver中pod的删除而删除，这样后面程序如果需要该pod信息可以从该storage中获取。同时该pod副本会通过异步清理goroutine在pod删除一小时后删除。`GetLocalPod`是从apiserver获取该node上所有的pod信息，该过程是调用kubernetes最多的地方，目前两个清理goroutine会每5min调用一次，调用量相对较小，对apiserver的负载影响不大。该模块也会在本地DB里缓存一份数据，便于在kubernetes pod删除后还可以拿到用户信息。

其次是resourceDB模块，该模块是用来持久化状态信息，该DB中记录了当前已分配的pod及其网络设备(networkResource)信息。每次请求/释放设备都会更新该DB。程序重新启动初始化完成之后，也会从resouceDB中恢复上次运行的数据。
除了基本的分配删除操作会更新该DB, terway还启动异步goroutine定期清理，保证异常情况下的最终一致性，该goroutine会从apiserve中获取所有pod信息和当前DB中的信息进行对比，如果对应的pod已经删除会先释放对应的网络设备，然后从DB中删除该记录。同时延迟清理可以实现Statefulset的Pod在更新过程中IP地址保持不变，

最重要的是`resouceManager`模块，该iterface封装了具体网络设备的操作，如下所示:

```java
// ResourceManager Allocate/Release/Pool/Stick/GC pod resource
// managed pod and resource relationship
type ResourceManager interface {
    Allocate(context *networkContext, prefer string) (types.NetworkResource, error)
    Release(context *networkContext, resID string) error
    GarbageCollection(inUseResList map[string]interface{}, expireResList map[string]interface{}) error
}
```

从其中三个method可以很明显的看出可以执行的的动作，每次CNI插件调用backendServer时， 就会调用ResoueceManager进行具体的分配释放操作。对于`ENIMultiIP`这种模式来说，具体的实现类是`eniIPResourceManager`：

```java
type eniIPResourceManager struct {
    pool pool.ObjectPool
}
```

其中只有pool一个成员函数，具体的实现类型是`simpleObjectPool`, 该pool维护了当前所有的ENI信息。当resouceManager进行分配释放网络设备的时候其实是从该pool中进行存取即可：

```java
func (m *eniIPResourceManager) Allocate(ctx *networkContext, prefer string) (types.NetworkResource, error) {
    return m.pool.Acquire(ctx, prefer)
}

func (m *eniIPResourceManager) Release(context *networkContext, resID string) error {
    if context != nil && context.pod != nil {
        return m.pool.ReleaseWithReverse(resID, context.pod.IPStickTime)
    }
    return m.pool.Release(resID)
}

func (m *eniIPResourceManager) GarbageCollection(inUseSet map[string]interface{}, expireResSet map[string]interface{}) error {
    for expireRes := range expireResSet {
        if err := m.pool.Stat(expireRes); err == nil {
            err = m.Release(nil, expireRes)
            if err != nil {
                return err
            }
        }
    }
    return nil
}
```

由上述代码可见，resouceManager实际操作的都是simpleObjectPool这个对象。　我们看看这个pool到底做了那些操作。首先初始化该pool:

```java
// NewSimpleObjectPool return an object pool implement
func NewSimpleObjectPool(cfg Config) (ObjectPool, error) {
    if cfg.MinIdle > cfg.MaxIdle {
        return nil, ErrInvalidArguments
    }

    if cfg.MaxIdle > cfg.Capacity {
        return nil, ErrInvalidArguments
    }

    pool := &simpleObjectPool{
        factory:  cfg.Factory,
        inuse:    make(map[string]types.NetworkResource),
        idle:     newPriorityQueue(),
        maxIdle:  cfg.MaxIdle,
        minIdle:  cfg.MinIdle,
        capacity: cfg.Capacity,
        notifyCh: make(chan interface{}),
        tokenCh:  make(chan struct{}, cfg.Capacity),
    }

    if cfg.Initializer != nil {
        if err := cfg.Initializer(pool); err != nil {
            return nil, err
        }
    }

    if err := pool.preload(); err != nil {
        return nil, err
    }

    log.Infof("pool initial state, capacity %d, maxIdle: %d, minIdle %d, idle: %s, inuse: %s",
        pool.capacity,
        pool.maxIdle,
        pool.minIdle,
        queueKeys(pool.idle),
        mapKeys(pool.inuse))

    go pool.startCheckIdleTicker()

    return pool, nil
}
```

可以看到在创建的时候会根据传入的config依次初始化各成员变量，　其中

- factory 成员用来分配网络设备，会调用ECS SDK进行分配资源，分配之后将信息存储在pool之中，具体的实现是`eniIPFactory`。
- inuse 存储了当前所有正在使用的networkResource
- idle 存储了当前所有空闲的networkResource, 即已经通过factory分配好，但是还未被某个pod实际使用。如果某个network resouce不再使用，也会归还到该idle之中。　通过这种方式，pool具备一定的缓充能力，避免频繁调用factory进行分配释放。idle为`priorityQeueu`类型，即所有空闲的networkResouce通过优先级队列排列，优先级队列的比较函数会比较`reverse`字段，`reverse`默认是入队时间，也就是该networkResouce的释放的时间，这样做能够尽量使一个IP释放之后不会被立马被复用。`reverse`字段对于一些statueSet的resouce也会进行一些特殊处理，因为statufulSet是有状态workload, 对于IP的释放也会特殊处理，保证其尽可能复用。
- maxIdle, minIdle 分别表示上述idle队列中允许的最大和最小个数，　minIdle是为了提供有一定的缓冲能力，但该值并不保证，最大是为了防止缓存过多，如果空闲的networkResouce太多没有被使用就会释放一部分，IP地址不止是节点级别的资源，也会占用整个vpc/vswitch/安全组的资源，太多的空闲可能会导致其他节点或者云产品分配不出IP。
- capacity 是该pool的容量，最大能分配的networkResouce的个数。该值可以自己指定， 但如果超过该ECS能允许的最大个数就会被设置成允许的最大个数。
- tokenCh 是个buffered channel, 容量大小即为上面capacity的值，被做token bucket。 pool初始化的时候会将其中放满元素，后面运行过程中中，只要能从该channel中读取到元素则意味着该pool还没有满。每次调用factory申请networkResouce之前会从该channel中读取一个元素，　每次调用factory释放networkDevice会从该channel中放入一个元素。

成员变量初始化完成之后会调用`Initializer`, 该函数会回调一个闭包函数，定义在`newENIIPResourceManager`中： 当程序启动时，resouceManager通过读取存储在本地磁盘也就是resouceDB中的信息获取当前正在使用的networkResouce，然后通过ecs获取当前所有eni设备及其ip, 依次遍历所有ip判断当前是否在使用，分别来初始化inuse和idle。这样可以保证程序重启之后可以重构内存中的pool数据信息。

然后会调用`preload`,该函数确保pool(idle)中有minIdle个空闲元素, 防止启动时大量调用factory。
最后会进行`go pool.startCheckIdleTicker()`　异步来goroutine中调用`checkIdle`定期查询pool(idle)中的元素是否超过maxIdle个元素，　如果超过则会调用factory进行释放。同时每次调用factory也会通过`notifyCh`来通知该goroutine执行检查操作。

pool结构初始化完成之后，resouceManager中所有对于networkResource的操作都会通过该pool进行，该pool在必要条件下再调用factory进行分配释放。

factory的具体实现是`eniIPFactory`, 用来调用ecs SDK进行申请释放eniIP, 并维护对应的数据结构。不同于直接使用eni设备，`ENIMultiIP`模式会为每个eni设备会有多个eniIP。eni设备是通过`ENI`结构体标识， eniIP通过`ENIIP`结构体标识。terway会为每个`ENI`创建一个goroutine, 该ENI上所有eniIP的分配释放都会在goroutine内进行，factory通过channel与该groutine通信， 每个goroutine对应一个接受channel `ipBacklog`，用于传递分配请求到该goroutine。 每次factory 需要创建(eniIPFactory.Create)一个eniIP时， 会一次遍历当前已经存在的`ENI`设备，如果该设备还有空闲的eniIP，就会通过该`ipBacklog` channel发送一个元素到该ENI设备的goroutine进行请求分配， 当goroutine将eniIP分配完毕之后通过factory 的`resultChan`通知factory, 这样factory就成功完成一次分配。 如果所有的ENI的eniIP都分配完毕，会首先创建ENI设备及其对应goroutine。因为每个ENI设备会有个主IP， 所以首次分配ENI不需要发送请求到`ipBacklog`, 直接将该主ip返回即可。对应的释放(Dispose)就是先释放eniIP， 等到只剩最后一个eniIP(主eniIP)时会释放整个ENI设备。对于所有ecs调用都会通过buffer channel进行流控，防止瞬间调用过大。

# 总结

总之，terway的整个实现，逻辑比较清晰，并且扩展性也较高。后期，可以比较方便地在此基础上做一些定制和运维支持，从而很好地融入公司的基础架构设施。