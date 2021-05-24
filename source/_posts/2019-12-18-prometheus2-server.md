---
title:      " Prometheus Server组件安装 "
date:       2019-12-18
banner_img: "https://image.my-blog.wang/header/header.jpg"
tags: [ Prometheus ]

---

这里 Prometheus Server 使用一个带 RBAC 权限的账号采集集群中现有监控信息（其实是从 cadvisor 获取）和节点信息。本次部署是基于比较新的 v2.11.1 版本，网上的一些教程还停留在比较早的版本，所以有些东西改动还是比较大的。

yaml文件地址：https://github.com/ILIKETWICE/prometheus-k8s

具体配置清单文件如下：

```bahs
├── prometheus-cm-alerts.yaml
├── prometheus-cm-config.yaml
├── prometheus-deploy.yaml
├── prometheus-ingress.yaml
├── prometheus-rbac.yaml
└── prometheus-svc.yaml
```

各个配置文件的作用如下：

**prometheus-cm-alerts.yaml：**该配置文件主要是针对 Alertmanager 定制的一些告警策略，这里暂时不需要 apply **；**

**prometheus-cm-config.yaml：** 该配置文件主要是 prometheus 主要的配置文件，prometheus 的配置基本都在此文件中定义；

**prometheus-rbac.yaml**：该文件主要定义了 Prometheus 容器访问 k8s apiserver 所需的 ServiceAccount、ClusterRole 以及 ClusterRoleBinding；

**prometheus-deploy.yaml**：该文件是 Prometheus 的主要的部署文件，里面定义了一些具体参数，可以根据实际集群的规格做相应的调整，个人建议 Prometheus 不要在业务机器上做部署，可以通过节点亲和等特性实现；

**prometheus-svc.yaml**：该文件定义了 Prometheus 的 Service，为了方便服务的暴露方式我选择了 NodePort 的方式，可以根据自己需求做相应的更改；

**prometheus-ingress.yaml**：该文件主要定义了 prometheus 服务的 Ingress 规则，可选；

配置清单文件准备好后就可以 apply 了：

```bash
[root@master-10 prometheus-server]# kubectl apply -f . --dry-run
configmap/prometheus-rule-config configured (dry run)
configmap/prometheus-config configured (dry run)
deployment.apps/prometheus-server configured (dry run)
ingress.extensions/prometheus-server-ingress configured (dry run)
serviceaccount/prometheus configured (dry run)
clusterrole.rbac.authorization.k8s.io/prometheus configured (dry run)
clusterrolebinding.rbac.authorization.k8s.io/prometheus configured (dry run)
service/prometheus configured (dry run)
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

需要注意的是上述信息是部署完成各个组件的部署信息，这里只关注 prometheus-server pod 即可；

Prometheus Server 到现在就可以说部署好了，当然我这里用的是 Deployment 在正式生产环境下还是建议用 SatefulSet ,保证后端存储的可靠性。