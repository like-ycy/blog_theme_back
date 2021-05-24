---
title:      "【Kubernetes系列】第5篇 Ingress Controller - Traefik组件介绍"
date:       2019-12-07
banner_img: "https://image.my-blog.wang/header/header.jpg"
tags: [ Kubernetes ]

---

#  1、概述

为了能够让Ingress资源能够工作，在Kubernetes集群中必须至少有一个运行中的ingress controller组件。也就是说如果在kubernetes集群中没有一个ingress controller组件，只是定义了ingress资源，其实并不会实现http、https协议的请求转发、负载均衡等功能。常见的ingress controller组件如下：

- Nginx
- Traefik
- Kong
- Istio
- HAProxy

关于上述的组件目前并没有详细的对比，后续我们在对每个组件都有一定的了解和使用的基础之上，可以给出一些详细的对比信息。本篇内容将主要介绍traefik组件的安装部署以及会通过一个具体的应用作演示。

#  2、Trafeik组件的安装部署

## 2.1 通过helm chart部署traefik

helm traefik chart包中包含了部署traefik组件的所需的资源，我们可以通过借助该组件进行快速部署traefik组件，以下是部署命令行信息：

```bash
> helm install --name inner-traefik --namespace kube-system\
  --set image=registry.docker.hankercloud.com/ingress-controller/traefik \
  --set serviceType=NodePort \
  stable/traefik
```

部署完成后，执行kubectl get pods -n kube-system命令，可以看到在kube-system的命名空间中已经存在名为 inner-traefik 的Pod。

## 2.2 RBAC配置

在kubernetes 1.6版本中引入了RBAC（Role Based Access Control）机制来更好的管理资源和API的访问。如果在集群中配置了RBAC，则需要授权Treafik使用Kubernetes的API，有两种方式来进行设置合适的策略：通过特定的命名空间进行角色绑定（RoleBinding）以及全局角色绑定（ClusterRoleBinding）。现在简单起见，我们直接使用ClusterRoleBinding，资源定义如下：

```yaml
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
rules:
  - apiGroups:
      - ""
    resources:
      - services
      - endpoints
      - secrets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-ingress-controller
subjects:
- kind: ServiceAccount
  name: traefik-ingress-controller
  namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
```

接下来我们执行如下命令创建资源并修改deployment的资源定义文件。

```bash
kubectl apply -f traefik-rbac.yml
kubectl edit deploy inner-traefik -n kube-system
```

执行完上述的操作之后，我们可以进行校验相关的资源已经正常启动。

```bash
kubectl logs $(kubectl get pods -n kube-system |grep traefik | awk '{print $1}') -n kube-system
```

## 2.3 负载均衡配置

由于我们使用的是Deployment部署的traefik组件，其Service Type为NodePort，通过kubectl get svc -n kube-system|grep traefik，可以看到端口映射关系，接下来我们在阿里云申请一个负载均衡的设备，然后进行相应的配置之后就完成了这一步操作。

另外一种替代方式是使用DaemonSet的方式部署traefik组件，设置主机端口和Pod实例端口的映射关系，也可以完成这一任务。

# 3、创建ingress资源并进行调试

接下来我们在kubernetes集群中创建一个ingress资源，由于我们之前已经在集群中部署了一个wordpress应用，资源定义文件如下：
```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: wordpress-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: blog.hankercloud.com
    http:
      paths:
      - path: /
        backend:
          serviceName: wordpress-test-wordpress
          servicePort: 80
```

完成上述的操作之后，我们在本地修改/etc/hosts文件，手动配置blog.hankercloud.com的域名解析记录，在浏览器地址栏输入 http://blog.hankercloud.com 就可以看到页面了，到此我们完成了traefik组件的安装部署及调试工作。
