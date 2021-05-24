---
title:      " Node-Expoter 组件安装 "
date:       2019-12-18
banner_img: "https://image.my-blog.wang/header/header.jpg"
tags: [ Prometheus ]

---

node-exporter 这里主要用来提供集群节点本身的信息的，包括 CPU、内存、硬盘、IO 等等信息。

yaml文件地址：https://github.com/ILIKETWICE/prometheus-k8s

具体配置清单文件如下：

```bash
├── node-exporter-ds.yaml
├── node-exporter-rbac.yaml
└── node-exporter-svc.yaml
```

各个配置文件的作用如下：

**node-exporter-rbac.yaml：**该文件主要定义了 node-exporter 容器访问 k8s apiserver 所需的 ServiceAccount、ClusterRole 以及 ClusterRoleBinding；

**node-exporter-ds.yaml**：该文件是 **node-exporter** 的主要的部署文件，里面定义了一些具体参数，可以根据实际集群的规格做相应的调整,这里用 DaemonSet,保证可以提供每个节点的信息；

**node-exporter-svc.yaml**：该文件定义了 **node-exporter** 的 Service，可以根据自己需求做相应的更改；

配置清单文件准备好后就可以 apply 了：

```bash
[root@master-10 node-exporter]# kubectl apply -f . --dry-run
daemonset.extensions/node-exporter configured (dry run)
serviceaccount/node-exporter configured (dry run)
service/node-exporter configured (dry run)
```

部署完成后就可以看到对应的 pod 运行起来了：

```bash
[root@master-10 prometheus-server]# kubectl get pods -n prometheus-huang
NAME                                 READY   STATUS    RESTARTS   AGE
alertmanager-59577dc4bf-qcnvw        1/1     Running   0          19h
kube-state-metrics-c8b4bcf8-jstlr    2/2     Running   0          43h
node-exporter-l4228                  1/1     Running   0          42h
node-exporter-qhqf2                  1/1     Running   0          42h
node-exporter-vjtdn                  1/1     Running   0          41h
prometheus-grafana-6d8676fb6-w8cvl   1/1     Running   0          24h
prometheus-server-784595847-kqznb    1/1     Running   0          19h
```

需要注意的是上述信息是部署完成各个组件的部署信息，这里只关注 node-exporter pod 即可；还有就是默认 pod 不会调度到 master 节点上的，需要添加容忍度才可以，具体如下：

```bash
spec:
      ...
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
```