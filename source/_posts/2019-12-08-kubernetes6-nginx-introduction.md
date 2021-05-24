---
title:      "【Kubernetes系列】第6篇 Ingress controller - nginx组件介绍"
date:       2019-12-08
banner_img: "https://image.my-blog.wang/header/header.jpg"
tags: [ Kubernetes ]

---

# 1、概述

在上一篇文章中我们介绍了如何通过helm进行安装部署traefik组件（[链接点这里](http://mp.weixin.qq.com/s?__biz=MzU5MTkyNzQ0MQ==&mid=2247483749&idx=1&sn=458f63caa65a8fbd33992b8414830388&chksm=fe26c09bc951498d183b2eb05d2f526e2e701e0432b51574ca932839f2146a94bc1e3a781bb7&scene=21#wechat_redirect)），文中还提到常用的ingress controller除了traefik还有Nginx、HAProxy、Kong等，在本篇文章中我们就介绍如何安装部署Nginx-ingress，只有在经过积累不同组件的使用经验之后，我们才能更好的比较其优劣，形成最佳实践。

# 2、安装部署

**2.1 通过helm查找nginx-ingress**

```shell
# step1: 通过helm查找nginx-ingress
> helm search nginx-ingress
> helm inspect stable/nginx-ingress
```

**2.2 镜像下载及上传**

部分企业由于服务器没有外网访问策略以及防火墙的原因无法获取国外Docker镜像，所以我们事先需要将所需镜像准备好，并上传到企业私有镜像仓库。

```
# step2: 镜像准备
> docker pull quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.25.1
> docker tag quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.25.1 registry.hankercloud.com/ingress-controller/nginx-ingress-controller:0.25.1
> docker push registry.hankercloud.com/ingress-controller/nginx-ingress-controller:0.25.1
>
> docker pull k8s.gcr.io/defaultbackend-amd64:1.5
> docker tag k8s.gcr.io/defaultbackend-amd64:1.5 registry.hankercloud.com/google_containers/defaultbackend-amd64:1.5
> docker push registry.hankercloud.com/google_containers/defaultbackend-amd64:1.5
```

**2.3 组件部署**

在上一篇博客中，我们是采用Deployment模式部署的traefik组件，这次我们采用DaemonSet的模式来部署nginx-ingress组件。

```
# step3: 组件部署
> helm install stable/nginx-ingress --name nginx-ingress --namespace=kube-system \
  --set fullnameOverride=nginx-ingress \
  --set controller.kind=DaemonSet \
  --set controller.daemonset.useHostPort=true \
  --set controller.metrics.enabled=true \
  --set controller.image.repository=registry.hankercloud.com/ingress-controller/nginx-ingress-controller \
  --set
  defaultBackend.image.repository=registry.hankercloud.com/google_containers/defaultbackend-amd64
# step4: 检查部署是否成功
> helm list> kubectl get all -n kube-system> kubectl logs $POD_NAME -n kube-system
```

**2.4 负载均衡配置及域名解析处理**

本次我们采用DaemonSet部署nginx-ingress组件，并且使用了主机的80和443接口用来分别接收http和https请求，我们将相应的域名解析到nginx-ingress Pod所在的主机IP之后，就可以通过域名来进行相应的域名访问了。

但上述配置方式无法做到高可用，当nginx-ingress的Pod实例故障或者其所在主机发生故障时，会导致相应的域名无法访问，所以建议在公有云购买负载均衡设备并配置相应的后端服务器列表以实现高可用的目的。

**2.5 安装调试**

在上文中我们通过helm部署了一个wordpress应用，本文我们继续通过该应用进行域名访问，在本机控制台输入 

```
> curl -i http://10.0.0.182 -H 'Host: blog.hankercloud.com'
```

如果看到有正常返回则说明部署成功。

# 3、企业场景及解决方案

**3.1 如何做内外网的隔离**

**Step1:** 我们首先部署了两个ingress组件，其中之一是接收内网访问请求，另外一个是接收外网访问请求，相应配置如下：

```yaml
# 内网nginx-ingress配置声明：
spec:
template:
    spec:
      containers:
- args:
- /nginx-ingress-controller
- --default-backend-service=kube-system/nginx-ingress-default-backend
- --election-id=ingress-controller-leader
- --ingress-class=nginx
- --configmap=kube-system/nginx-ingress-controller
```
```yaml
# 外网nginx-ingress配置声明：
spec:
template:
    spec:
      containers:
- args:
- /nginx-ingress-controller
- --default-backend-service=kube-system/nginx-ingress-external-default-backend
- --election-id=ingress-controller-leader
- --ingress-class=nginx-external
- --configmap=kube-system/nginx-ingress-external-controller
```

两者的主要区别在于参数 --ingress-class 设置的值是不一样的。

**Step2:** 对于需要暴露到公网的域名，修改其ingress的定义，相应配置参考如下：

```yaml
metadata:
  name: www
  annotations:
    kubernetes.io/ingress.class: "nginx-external"
```

**Step3:** 检查是否配置成功，执行 

```shell
kubectl exec ${POD_NAME} -n kube-system cat /etc/nginx/nginx.conf
```

查看配置文件中是否已经包含外网域名的相关配置，并在本地进行测试验证。