---
title:      " 运维面试题系列--Docker "
date:       2021-03-03
banner_img: "https://gitee.com/like-ycy/images/raw/master/blog/header.jpg"
tags: [ 面试 ]


---

# Docker 面试问题



### 1、docker网络模式

```
bridge：这是Docker默认的网络驱动，此模式会为每一个容器分配Network Namespace和设置IP等，并将容器连接到一个虚拟网桥上。如果未指定网络驱动，这默认使用此驱动。
host：此网络驱动直接使用宿主机的网络。
none：此驱动不构造网络环境。采用了none 网络驱动，那么就只能使用loopback网络设备，容器只能使用127.0.0.1的本机网络。
overlay：此网络驱动可以使多个Docker daemons连接在一起，并能够使用swarm服务之间进行通讯。也可以使用overlay网络进行swarm服务和容器之间、容器之间进行通讯，
macvlan：此网络允许为容器指定一个MAC地址，允许容器作为网络中的物理设备，这样Docker daemon就可以通过MAC地址进行访问的路由。对于希望直接连接网络网络的遗留应用，这种网络驱动有时可能是最好的选择。
```





## 镜像相关

### 1、如何批量清理临时镜像文件？

```
docker rmi $(docker images -q -f danging=true)
```

### 2、如何查看镜像支持的环境变量？

```
docker run IMAGE env
```

### **3、**本地的镜像文件都存放在哪里**？**

```
与Docker相关的本地资源存放在/var/lib/docker/目录下，其中container目录存放容器信息， graph目录存放镜像信息，aufs目录下存放具体的镜像底层文件。
```

### 4、构建Docker镜像应该遵循哪些原则？

```
整体原则上，尽量保持镜像功能的明确和内容的精简，要点包括：  
# 尽量选取满足需求但较小的基础系统镜像，建议选择debian:wheezy镜像，仅有86MB大小  
# 清理编译生成文件、安装包的缓存等临时文件  
# 安装各个软件时候要指定准确的版本号，并避免引入不需要的依赖  
# 从安全的角度考虑，应用尽量使用系统的库和依赖  
# 使用Dockerfile创建镜像时候要添加.dockerignore文件或使用干净的工作目录
```

## 容器相关

### 1、容器退出后，通过docker ps 命令查看不到，数据会丢失么？

```
容器退出后会处于终止（exited）状态，此时可以通过 docker ps -a 查看，其中数据不会 丢失，还可以通过docker start 来启动，只有删除容器才会清除数据。
```

### 2、如何停止所有正在运行的容器？

```
docker kill $(docker ps -q)
```

### 3、如何清理批量后台停止的容器？

```
docker rm $（docker ps -a -q）
```

### 4、如何获取某个容器的PID信息?

```
docker inspect --format '{{ .State.Pid }}' <CONTANINERID or NAME>
```

### 5、如何获取某个容器的IP地址?

```
docker  inspect --format '{{ >NetworkSettings.IPAddress }}' <CONTANINERID or NAME>
```

### 6、如何给容器指定一个固定IP地址，而不是每次重启容器IP地址都会变？

```
目前Docker并没有提供直接的对容器IP地址的管理支持，可以在网上查找容器网络配置创建点 对点连接的案例，来手动配置容器的静态IP。或者在容器启动后，再手动进行修改。
```

### 7、如何临时退出一个正在交互的容器的终端，而不终止它？

```
按Ctrl+p，后按Ctrl+q，如果按Ctrl+c会使容器内的应用进程终止，进而会使容器终止。
```

### 8、很多应用容器都是默认后台运行的，怎么查看它们的输出和日志信息？

```
docker logs 容器名称或容器id
```

### 9、使用docker port 命令映射容器的端口时，系统报错Error: No public port ‘80’ published for …，是什么意思？

```
创建镜像时Dockerfile要指定正确的EXPOSE的端口，容器启动时指定PublishAllport=true
```

### 10、可以在一个容器中同时运行多个应用进程吗？

```
一般不推荐在同一个容器内运行多个应用进程，如果有类似需求，可以通过额外的进程管理机制， 比如supervisord来管理所运行的进程
```

### 11、如何控制容器占用系统资源（CPU，内存）的份额？

```
在使用docker create命令创建容器或使用docker run 创建并运行容器的时候， 可以使用-c|–cpu-shares[=0]参数来调整同期使用CPU的权重， 使用-m|–memory参数来调整容器使用内存的大小。
```

## 仓库相关

### 1、仓库（Repository）、注册服务器（Registry）、注册索引（Index）有何关系？

```
首先，仓库是存放一组关联镜像的集合，比如同一个应用的不同版本的镜像，注册服务器是存放 实际的镜像的地方，注册索引则负责维护用户的账号，权限，搜索，标签等管理。注册服务器利 用注册索引来实现认证等管理。
```

### 2 、从非官方仓库（如：dl.dockerpool.com）下载镜像的时候，有时候会提示“Error：Invaild registry endpoint https://dl.docker.com:5000/v1/…”?

```
Docker 自1.3.0版本往后以来，加强了对镜像安全性的验证，需要手动添加对非官方仓库的信任。  DOCKER_OPTS=”–insecure-registry dl.dockerpool.com:5000”  重启docker服务
```

### 3.docker pull 某一个镜像很慢，比如说centos. 这种情况是因为docker官方的镜像库是国外，我们应该配置国内的镜像库或者加速器。

```
加速器可以用阿里云镜像加速器，进入阿里云网站在控制台可以看到，根据提示来即可。

配置文件 /etc/docker/daemon.json 添加如下命令

DOCKER_NETWORK_OPTIONS="--registry-mirror=https://....."   地址可以为国内镜像地址，也可为公司内网镜像仓库地址。
```



### 4.docker push镜像时碰到的443问题，这是因为docker默认会走ssl加密的。

```
如果是公司内网的，直接在/etc/sysconfig/docker-network 里面添加如下配合

DOCKER_NETWORK_OPTIONS=“--insecure-registry xxx-ip”
```



### 5.碰到网络问题，无法pull镜像，命令行指定http_proxy无效，怎么办？

```
在Docker配置文件中添加export http_proxy="http://<PROXY_HOST>:<PROXY_PORT>"， 之后重启Docker服务即可。
```

## 配置相关

### 1、Docker的配置文件放在那里。如何修改配置？

```
Ubuntu系统下Docker的配置文件是/etc/default/docker，CentOS系统配置文件存放在/etc/sysconfig/docker
```

### 2、如何更改Docker的默认存储设置？

```
Docker的默认存放位置是/var/lib/docker,如果希望将Docker的本地文件存储到其他分区，可以使用Linux软连接的方式来做。
```

## Docker与虚拟化

### 1、Docker与LXC（Linux Container）有何不同？

```
LXC利用Linux上相关技术实现容器，Docker则在如下的几个方面进行了改进：

移植性：通过抽象容器配置，容器可以实现一个平台移植到另一个平台；  镜像系统：基于AUFS的镜像系统为容器的分发带来了很多的便利，同时共同的镜像层只需要存储一份，实现高效率的存储；  版本管理：类似于GIT的版本管理理念，用户可以更方面的创建、管理镜像文件；  仓库系统：仓库系统大大降低了镜像的分发和管理的成本；  周边工具：各种现有的工具（配置管理、云平台）对Docker的支持，以及基于Docker的Pass、CI等系统，让Docker的应用更加方便和多样化。
```

