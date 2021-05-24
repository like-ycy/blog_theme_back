---
title:      " mesos 以容器方式启动,拉取镜像失败问题 "
date:       2020-01-01
banner_img: "https://image.my-blog.wang/header/header.jpg"
tags: [ Mesos ]


---

为了方便快速部署,将 mesos、marathon进行了容器化部署, 但是容器化完后发现在`marathon` 上创建应用一直创建不成功

**分析过程**

因为是容器化部署的mesos-slave和marathon,是不是因为没有找到证书和登录信息导致的,随后在mesos-salve容器手动的进行`docker login` 操作并将证书进行挂载到对应目录,手工执行命令docker -H unix:///var/run/docker.sock pull registry.cn-beijing.aliyuncs.com/xxxx/xxxxx-service:12984 能正常下载仓库镜像,但通过marathon创建应用问题依旧,此时我们查看了一下日志发现一些踪迹

**mesos-slave 错误信息**

```bash
Failed to launch container: Failed to run 'docker -H unix:///var/run/docker.sock pull registry.cn-beijing.aliyuncs.com/xxxx/xxxxx-service:12984': exited with status 1; stderr='Error response from daemon: pull access denied for registry.cn-beijing.aliyuncs.com/xxxxx/xxxxx-service, repository does not exist or may require 'docker login': denied: requested access to the resource is denied '
```

镜像仓库报错很明显,认证不通过,通过查找`marathon` 官方文档,我们发现官方说明在使用`Private Docker Registry` 时候需要额外做一些配置,参考[marathon使用私有仓库配置](https://mesosphere.github.io/marathon/docs/native-docker-private-registry.html) ,配置简单的步骤如下,所有`slave`节点和`marathon`节点都需要配置

- 宿主机手工进行docker login

```bash
docker login 
```


- 登录信息会保存在/root/.docker 目录下,将其进行打包

```bash
cd /root
tar -cvf docker.tar.gz .docker/
```

- 将打包的文件复制到所有节点目录,**注意:** 放置在一个公共目录,因为marathon和mesos-slave都需要调用到

```bash
scp  docker.tar.gz  所有节点:/一个公共目录
```

- 容器启动marathon 和mesos-slave时候进行挂载对应公共目录

```bash
#mesos-slave
docker run -v /tmp/docker.tar.gz:/tmp/docker.tar.gz  .....  mesos-salve
#marathon
docker run -v /tmp/docker.tar.gz:/tmp/docker.tar.gz  .....  marathon
```


- 创建应用时候,配置urls参数

```json
"fetch": [
  {
    "uri": "file:///etc/docker.tar.gz"
  }
]
```

应用启动后,我们登录到mesos-slave 容器上查看资源stderr日志可以发现,资源节点如何下载和使用docker.tar.gz 包，这里就不贴出日志信息啦。

因为我使用的图形化界面配置，mesos版本为1.5.0，配置图如下

![](https://gitee.com/like-ycy/images/raw/master/blog/2020-01-01/1.png)

![](https://gitee.com/like-ycy/images/raw/master/blog/2020-01-01/2.png)

![](https://gitee.com/like-ycy/images/raw/master/blog/2020-01-01/3.png)